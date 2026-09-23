# Login and Hub Token Setup (cloud-services)

How to configure GUI login (NetID allowlist + one shared team password) and hub device tokens on
the production cloud service, how to verify it, and how to rotate or revoke access later.

Written for humans and agents. Values in `<ANGLE_BRACKETS>` are placeholders you must replace.
Every step ends with an **Expected result**; stop and troubleshoot if it does not match.

## Rules (read first)

- Never paste the team password, the JWT secret, or device tokens into chat, commits, PR text,
  issues, or logs. The password hash is less sensitive but still keep it private.
- Secrets live only in the backend env file on the server (and each Pi's `.env` for its own token).
  Both are gitignored; never `git add` them.
- Configure the server **before** the new cloud-services code is deployed. With
  `ENVIRONMENT=production` the service refuses to start if any value below is missing or still a
  development default, and deploys happen automatically when the hyperloop-gui PR merges.
- The old per-user passwords (`users.json`) and the old credential generator are retired. The
  Gmail app password that was committed with the generator must be revoked in that Google
  account (Google Account > Security > App passwords) if that has not been done.

## How login works now

- A person signs in with their **NetID** and the **team password**. The NetID must be on the
  allowlist. The JWT carries the NetID, so actions stay attributable to a person.
- "View Only Mode" needs no credentials and cannot send any command (write, restart, close, flash).
- Failed logins are limited per NetID (5 per 5 minutes) and per client IP (20 per 5 minutes).
- Removing a NetID from the allowlist ends that person's existing sessions on their next request.
- Development (no production settings): NetID `dev`, password `hyperloop-dev`; hubs connect with
  `dev-token-rpi-bridge-01` / `dev-token-rpi-bridge-02`.

## Setup

### Step 1: Find the backend's env file

On the EC2 host (SSH in as the deploy user):

```bash
systemctl cat backend
```

Look at `WorkingDirectory=` and for `ENV_FILE=` in `Environment=`/`ExecStart=`, or an
`EnvironmentFile=`. cloud-services reads the file named by `ENV_FILE`, relative to the working
directory, and falls back to `.env`.

**Expected result:** you know the path of the env file, referred to below as `<ENV_FILE>`
(typically `/home/ubuntu/cloud-services/.env.production`). If you cannot find one, create
`/home/ubuntu/cloud-services/.env.production` and make sure the unit sets
`Environment=ENV_FILE=.env.production`.

### Step 2: Choose the team password and hash it

Pick a long passphrase (for example four or more random words; at most 72 bytes). Hash it on the
server with the backend's virtualenv. The password is typed at a hidden prompt, so it never
appears in shell history:

```bash
cd /home/ubuntu/cloud-services
venv/bin/python -c "import bcrypt, getpass; p = getpass.getpass('Team password: ').encode(); print(bcrypt.hashpw(p, bcrypt.gensalt(12)).decode())"
```

**Expected result:** one line starting with `$2b$12$`. This is `<TEAM_PASSWORD_HASH>`. Give the
passphrase itself to the team in person or through a private channel.

### Step 3: Generate the JWT secret

```bash
venv/bin/python -c "import secrets; print(secrets.token_urlsafe(48))"
```

**Expected result:** a random string of about 64 characters, `<JWT_SECRET>`.

### Step 4: Create the NetID allowlist

```bash
nano /home/ubuntu/allowed_netids.txt
chmod 600 /home/ubuntu/allowed_netids.txt
```

One NetID per line, lowercase, without `@cornell.edu`. `#` starts a comment:

```text
# Electrical subteam
abc123
xyz789  # team lead
```

**Expected result:** the file lists every member who should be able to send commands.
`viewer`, `admin` and `root` are ignored if listed. (Alternative: put a comma-separated list in
`ALLOWED_NETIDS=` instead of using a file.)

### Step 5: Generate one device token per hub

Decide the hub IDs first. They must match `HUB_ID` in each Pi's `.env`
(see `.claude/rpi-hub-setup.md`). Example IDs: `rpi-lab-01`, `rpi-lab-02`, `rpi-lab-03`,
`rpi-cellular-01`. Existing Pis can keep their current IDs (`rpi-bridge-01`, `rpi-bridge-02`).

Run once per hub:

```bash
venv/bin/python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Build the value as comma-separated `hub-id:token` pairs:

```text
DEVICE_TOKENS=rpi-lab-01:<TOKEN_1>,rpi-lab-02:<TOKEN_2>,rpi-lab-03:<TOKEN_3>,rpi-cellular-01:<TOKEN_4>
```

**Expected result:** every hub has its own token. Record which token belongs to which Pi somewhere
private; each Pi needs its token as `DEVICE_TOKEN`. Every hub listed here appears on the GUI
dashboard (offline until it connects).

### Step 6: Write the env file

Edit `<ENV_FILE>` and set (keep any other existing lines such as `CORS_ORIGINS`):

```bash
ENVIRONMENT=production
JWT_SECRET_KEY=<JWT_SECRET>
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=480
# Single quotes keep the $ characters of the hash literal
TEAM_PASSWORD_HASH='<TEAM_PASSWORD_HASH>'
ALLOWED_NETIDS_FILE=/home/ubuntu/allowed_netids.txt
DEVICE_TOKENS=<VALUE_FROM_STEP_5>
# Only if port 8080 is not reachable from the internet (all traffic comes through Caddy/Cloudflare):
TRUST_PROXY_HEADERS=true
```

Then:

```bash
chmod 600 <ENV_FILE>
```

Remove `USERS_FILE_PATH` if present (no longer used). Old `DEVICE_TOKEN_RPI_BRIDGE_01/02` lines
still work, but move those hubs into `DEVICE_TOKENS` and delete the old lines.

**Expected result:** the file contains every key above with real values; no `dev-token`
values remain.

### Step 7: Restart and check the service

```bash
sudo systemctl restart backend
sleep 3
systemctl is-active backend
journalctl -u backend -n 50 --no-pager
```

**Expected result:** `active`, and no `Invalid production configuration` in the log. If that
error appears, it lists exactly which setting is missing or invalid.

### Step 8: Verify

Set the API base (use `http://127.0.0.1:8080` when testing on the server itself):

```bash
API=https://api.cornellhyperloop.com
```

1. Login works (enter your NetID and the team password when prompted):

   ```bash
   read -p "NetID: " NETID; read -s -p "Team password: " PW; echo
   curl -s -o /dev/null -w "%{http_code}\n" -X POST "$API/auth/login" \
     -H 'content-type: application/json' -d "{\"username\":\"$NETID\",\"password\":\"$PW\"}"
   ```

   **Expected result:** `200`

2. A wrong password is rejected:

   ```bash
   curl -s -o /dev/null -w "%{http_code}\n" -X POST "$API/auth/login" \
     -H 'content-type: application/json' -d "{\"username\":\"$NETID\",\"password\":\"wrong\"}"
   ```

   **Expected result:** `401`. After 5 failures in 5 minutes the same NetID gets `429`, even with
   the right password, until the window passes. Test that with a throwaway NetID rather than your
   own.

3. View-only sessions cannot send commands:

   ```bash
   VIEWER=$(curl -s -X POST "$API/auth/login-viewer" | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")
   curl -s -o /dev/null -w "%{http_code}\n" -X POST "$API/api/hubs/<HUB_ID>/commands/restart" \
     -H "Authorization: Bearer $VIEWER" -H 'content-type: application/json' -d '{"portId":"x"}'
   ```

   **Expected result:** `403`

4. Configured hubs are listed:

   ```bash
   curl -s "$API/api/hubs" -H "Authorization: Bearer $VIEWER"
   ```

   **Expected result:** one entry per hub in `DEVICE_TOKENS`, `"connected": false` until that Pi
   is set up.

5. In the GUI, sign in with your NetID and the team password.

Clean up the shell: `unset PW NETID VIEWER`.

Once login works, delete the old users file if it exists: `rm -f /home/ubuntu/users.json`.

## Maintenance

| Task | Steps | Takes effect |
|---|---|---|
| Add or remove a member | Edit `allowed_netids.txt` | Next request (no restart); removed members are logged out |
| Rotate the team password | Step 2, replace `TEAM_PASSWORD_HASH`, restart `backend` | New logins; existing sessions last until they expire (8 h) |
| Log everyone out now | Replace `JWT_SECRET_KEY` (step 3), restart `backend` | Immediately |
| Add a hub | Generate a token (step 5), append to `DEVICE_TOKENS`, restart `backend`, put the token in the Pi's `.env` | After restart |
| Revoke a hub (lost/stolen Pi) | Remove its pair from `DEVICE_TOKENS`, restart `backend` | Immediately; the hub is rejected (`ws_rejected` in its log) |

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `backend` fails to start with `Invalid production configuration` | The message names the missing or invalid setting; fix it in `<ENV_FILE>` |
| Everyone gets `401` with the right password | Hash copied incorrectly (must start with `$2b$` and be in single quotes), or the service reads a different env file (step 1) |
| One person gets `401` | Their NetID is not in the allowlist, or typo in the file |
| Everyone gets `429` | Without `TRUST_PROXY_HEADERS=true` behind a proxy, all users share the proxy's IP (20 failures per 5 minutes locks everyone out). Set it only if port 8080 is not publicly reachable |
| A hub never shows online | Its `HUB_ID`/`DEVICE_TOKEN` pair does not match `DEVICE_TOKENS`; see `.claude/rpi-hub-setup.md` |
