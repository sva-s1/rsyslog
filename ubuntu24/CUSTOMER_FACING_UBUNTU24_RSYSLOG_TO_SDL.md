# Customer Facing: Forward Syslog to SentinelOne Data Lake on Ubuntu 24.04 with rsyslog

This customer-facing how-to configures a native Ubuntu 24.04 LTS host to receive syslog over UDP/TCP `514` and forward each raw syslog message to SentinelOne Data Lake / DataSet (SDL) with the HEC raw ingest API.

The deployment uses the Ubuntu-provided `rsyslog` package plus a small `omprog` sender script. No container runtime, agent, or custom-built rsyslog module is required.

> [!NOTE]
> SentinelOne Data Lake, SentinelOne DataSet, and SDL are used interchangeably in some environments. This guide uses **SDL** for brevity.

## Why this guide uses an `omprog` sender script

Some examples use rsyslog HTTP output modules such as `omhttp` or HEC-specific output modules. On a stock Ubuntu 24.04 LTS installation, the needed rsyslog HTTP output module is not available in the default Ubuntu repositories in a clean, supported way. Building third-party rsyslog modules adds packaging, upgrade, and support risk.

This guide uses rsyslog's built-in `omprog` module to pipe each raw message to a short shell script. The script posts the message to SDL HEC with `curl`. This is an intentional Ubuntu 24.04 compatibility workaround: it keeps the deployment native, transparent, and easy to troubleshoot while avoiding custom rsyslog module builds.

## Tested baseline

Validated with Ubuntu `24.04.4 LTS`, rsyslog `8.2312.0`, inbound syslog UDP/TCP `514`, SDL HEC raw ingest such as `https://ingest.us1.sentinelone.net/services/collector/raw?sourcetype=syslog`, and SDL query API such as `https://xdr.us1.sentinelone.net/api/query`.

Your SDL region may use different hostnames. Use the ingest and query/API hostnames from your SentinelOne tenant.

## Prerequisites

On the Ubuntu 24.04 host:

- root or `sudo` access
- outbound HTTPS access to the SDL ingest endpoint on TCP `443`
- inbound UDP/TCP `514` from devices that will send syslog
- SDL HEC write key / Log Access Key with write permissions
- SDL read/query key / Log Access Key with read permissions, for validation
- SDL ingest hostname, query hostname, and desired sourcetype

## 1. Install packages

```bash
sudo apt-get update
sudo apt-get install -y rsyslog curl netcat-openbsd iproute2 ca-certificates
rsyslogd -v | head -n 1
sudo systemctl enable rsyslog
```

Ubuntu 24.04 normally reports rsyslog `8.2312.0`.

## 2. Store SDL settings in a root-protected environment file

Create `/etc/sdl/sdl.env`:

```bash
sudo install -d -m 0750 -o root -g syslog /etc/sdl
sudo tee /etc/sdl/sdl.env > /dev/null <<'EOF'
SDL_URL=https://<YOUR_INGEST_HOSTNAME>/services/collector/raw?sourcetype=<YOUR_SOURCETYPE>
SDL_PASS=<YOUR_HEC_WRITE_KEY>
EOF
sudo chown root:syslog /etc/sdl/sdl.env
sudo chmod 0640 /etc/sdl/sdl.env
```

Replace:

| Placeholder | Example |
|---|---|
| `<YOUR_INGEST_HOSTNAME>` | `ingest.us1.sentinelone.net` |
| `<YOUR_SOURCETYPE>` | `syslog` |
| `<YOUR_HEC_WRITE_KEY>` | SDL Log Access Key with write permission |

> [!TIP]
> Why `/etc/sdl/sdl.env`?
> - The key stays out of the executable script.
> - The file is protected by root ownership and mode `0640`.
> - rsyslog on Ubuntu drops privileges to the `syslog` user, so group read permission is required.
>
> Customers may place this file elsewhere if their standard requires it. If moved, update `ENV_FILE` in the sender script and the AppArmor rule below.

Verify permissions without printing the secret value:

```bash
sudo stat -c '%U:%G:%a %n' /etc/sdl/sdl.env
sudo sh -c '. /etc/sdl/sdl.env; echo "SDL_URL=$SDL_URL"; echo "SDL_PASS length=${#SDL_PASS}"'
```

Expected permission output:

```text
root:syslog:640 /etc/sdl/sdl.env
```

## 3. Create the SDL sender script

Create `/usr/local/bin/sdl-sender.sh`:

```bash
sudo tee /usr/local/bin/sdl-sender.sh > /dev/null <<'EOF'
#!/bin/sh
# Forward syslog messages from rsyslog omprog to SentinelOne Data Lake HEC.
# One raw syslog message is read per line from stdin and POSTed to SDL.

LOG_FILE="/var/log/sdl-sender.log"
ENV_FILE="/etc/sdl/sdl.env"

if [ -r "$ENV_FILE" ]; then
    . "$ENV_FILE"
fi

if [ -z "${SDL_URL:-}" ] || [ -z "${SDL_PASS:-}" ]; then
    printf 'ERROR missing SDL_URL or SDL_PASS in %s\n' "$ENV_FILE" >> "$LOG_FILE"
    exit 1
fi

while IFS= read -r msg; do
    # Keep a local copy during validation. Remove this line after rollout if local copies are not desired.
    printf 'EVENT %s\n' "$msg" >> "$LOG_FILE"

    H1='Authorization:'
    H2='Bearer'
    AUTH_HEADER="$H1 $H2 $SDL_PASS"
    curl -sS -m 10 -X POST "$SDL_URL" \
        -H "$AUTH_HEADER" \
        -H "Content-Type: text/plain" \
        --data-raw "$msg" >> "$LOG_FILE" 2>&1
    printf '\n' >> "$LOG_FILE"
done
EOF

sudo chown root:root /usr/local/bin/sdl-sender.sh
sudo chmod 0755 /usr/local/bin/sdl-sender.sh
sudo sh -n /usr/local/bin/sdl-sender.sh
```

Create the local sender log with permissions that work after rsyslog drops privileges:

```bash
sudo touch /var/log/sdl-sender.log
sudo chown syslog:adm /var/log/sdl-sender.log
sudo chmod 0664 /var/log/sdl-sender.log
```

## 4. Configure rsyslog to listen and forward to SDL

Create `/etc/rsyslog.d/10-sdl.conf`:

```bash
sudo tee /etc/rsyslog.d/10-sdl.conf > /dev/null <<'EOF'
# Receive syslog over UDP and TCP, then forward raw messages to SDL.

module(load="imudp")
module(load="imtcp")
module(load="omprog")

template(name="rawmsg_newline" type="string" string="%rawmsg%\n")

action(
    type="omprog"
    binary="/usr/local/bin/sdl-sender.sh"
    template="rawmsg_newline"
)

input(type="imudp" port="514")
input(type="imtcp" port="514")
EOF
```

> [!TIP]
> If you only need to forward the host's own local logs and do **not** need to receive network syslog, remove both `input(...)` lines and the `imudp` / `imtcp` module lines.

Validate syntax:

```bash
sudo rsyslogd -N1
```

Expected ending:

```text
rsyslogd: End of config validation run. Bye.
```

## 5. Add AppArmor permissions on Ubuntu 24.04 if needed

Ubuntu 24.04 commonly runs rsyslog under an AppArmor profile. In testing, the default profile blocked `omprog` from executing a script under `/usr/local/bin` until a narrow allow-list was added.

Create a local include only when the rsyslog AppArmor profile exists:

```bash
if [ -d /etc/apparmor.d/rsyslog.d ] && [ -f /etc/apparmor.d/usr.sbin.rsyslogd ]; then
  sudo tee /etc/apparmor.d/rsyslog.d/sdl-sender > /dev/null <<'EOF'
# Allow rsyslog omprog to execute the SDL sender and curl to HEC.
/usr/local/bin/sdl-sender.sh rix,
/etc/sdl/sdl.env r,
/usr/bin/curl rix,
/bin/sh ix,
/bin/dash ix,
/etc/ssl/** r,
/usr/share/ca-certificates/** r,
EOF

  sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.rsyslogd
fi
```

> [!IMPORTANT]
> Do not disable AppArmor for this integration. The narrow include above is easier to audit and preserves Ubuntu's default security posture.

## 6. Open the firewall if this host receives remote syslog

If UFW is enabled:

```bash
sudo ufw allow 514/udp
sudo ufw allow 514/tcp
sudo ufw reload
```

Also ensure any network firewall permits source devices to reach the Ubuntu host on UDP/TCP `514`, and the Ubuntu host to reach SDL ingest on TCP `443`.

## 7. Start rsyslog and confirm listeners

```bash
sudo systemctl restart rsyslog
sudo systemctl is-enabled rsyslog
sudo systemctl is-active rsyslog
sudo ss -ulnp | grep ':514'
sudo ss -tlnp | grep ':514'
```

Expected service state:

```text
enabled
active
```

Expected listener shape:

```text
udp   UNCONN ... 0.0.0.0:514 ... users:(("rsyslogd",...))
tcp   LISTEN ... 0.0.0.0:514 ... users:(("rsyslogd",...))
```

## 8. Test SDL HEC connectivity directly

Before testing rsyslog, verify the host can post to SDL HEC:

```bash
RUN_ID="sdl-direct-test-$(date +%s)"
sudo sh -c '. /etc/sdl/sdl.env; H1="Authorization:"; H2="Bearer"; AUTH_HEADER="$H1 $H2 $SDL_PASS"; curl -sS -m 10 -X POST "$SDL_URL" \
  -H "$AUTH_HEADER" \
  -H "Content-Type: text/plain" \
  --data-raw "'"$RUN_ID"' direct HEC test from $(hostname)"'
echo "$RUN_ID"
```

Expected response:

```json
{"text":"Success","code":0}
```

> [!WARNING]
> If this fails, fix the SDL URL, write key, DNS, proxy, or outbound firewall issue before continuing.

## 9. Send test syslog messages

Use one run ID so the messages are easy to find in SDL:

```bash
RUN_ID="sdl-rsyslog-test-$(date +%s)"
echo "RUN_ID=$RUN_ID"
```

Local UDP test:

```bash
echo "<34>$(date '+%b %e %T') localhost test[localudp]: $RUN_ID hello via udp localhost to SDL" | \
  nc -u -w 2 127.0.0.1 514
```

Local TCP test:

```bash
echo "<34>$(date '+%b %e %T') localhost test[localtcp]: $RUN_ID hello via tcp localhost to SDL" | \
  nc -w 2 127.0.0.1 514
```

Remote sender test from another host, replacing `<UBUNTU_SYSLOG_HOST_IP>`:

```bash
echo "<34>$(date '+%b %e %T') remotehost test[udp]: $RUN_ID hello via udp remote to SDL" | \
  nc -u -w 2 <UBUNTU_SYSLOG_HOST_IP> 514

echo "<34>$(date '+%b %e %T') remotehost test[tcp]: $RUN_ID hello via tcp remote to SDL" | \
  nc -w 2 <UBUNTU_SYSLOG_HOST_IP> 514
```

## 10. Check local sender evidence

On the Ubuntu host:

```bash
sudo grep "$RUN_ID" /var/log/sdl-sender.log
sudo grep -A1 "$RUN_ID" /var/log/sdl-sender.log
```

Expected for each message:

```text
EVENT <34>... sdl-rsyslog-test-... hello via ... to SDL
{"text":"Success","code":0}
```

> [!TIP]
> If you see the `EVENT` line but not `{"text":"Success","code":0}`, rsyslog received the message but SDL delivery failed. Check the curl output in `/var/log/sdl-sender.log`.

## 11. Query SDL for test messages

Use the SDL query API. The read/query key is sent in the JSON body as `token`.

```bash
READ_KEY='<YOUR_SDL_READ_KEY>'
QUERY_URL='https://<YOUR_QUERY_HOSTNAME>/api/query'
RUN_ID='<RUN_ID_FROM_TEST_STEP>'

curl -sS -m 30 -X POST "$QUERY_URL" \
  -H 'Content-Type: application/json' \
  -d "$(printf '{"token":"%s","queryType":"log","filter":"%s","startTime":"30m","maxCount":10,"pageMode":"tail","priority":"low"}' "$READ_KEY" "$RUN_ID")"
```

A successful response has HTTP status `200`, JSON field `"status":"success"`, and one `matches[]` entry for each test message.

> [!NOTE]
> SDL query indexing can lag ingest by a short time. If the local sender log shows HEC success but the query returns zero or fewer matches than expected, wait 10-30 seconds and run the same query again before troubleshooting the configuration.

Minimal pretty-print helper:

```bash
curl -sS -m 30 -X POST "$QUERY_URL" \
  -H 'Content-Type: application/json' \
  -d "$(printf '{"token":"%s","queryType":"log","filter":"%s","startTime":"30m","maxCount":10,"pageMode":"tail","priority":"low"}' "$READ_KEY" "$RUN_ID")" |
python3 -c 'import json,sys; d=json.load(sys.stdin); print("status=", d.get("status")); print("match_count=", len(d.get("matches", []))); [print(m.get("message", "").strip()) for m in d.get("matches", [])]'
```

Expected output for four tests:

```text
status= success
match_count= 4
<34>... localudp ...
<34>... localtcp ...
<34>... remotehost test[udp] ...
<34>... remotehost test[tcp] ...
```

## 12. Restart validation

Restart `rsyslog` and verify that the service returns to the expected enabled and active state with UDP/TCP listeners still present:

```bash
sudo systemctl restart rsyslog
sudo systemctl is-enabled rsyslog
sudo systemctl is-active rsyslog
sudo ss -ulnp | grep ':514'
sudo ss -tlnp | grep ':514'
```

Then send at least one fresh test message and confirm both local HEC success and SDL query visibility.

## Troubleshooting checklist

### rsyslog configuration does not validate

```bash
sudo rsyslogd -N1
sudo journalctl -u rsyslog -n 80 --no-pager
```

### rsyslog is not listening on port 514

```bash
sudo ss -ulnp | grep ':514' || true
sudo ss -tlnp | grep ':514' || true
sudo journalctl -u rsyslog -n 80 --no-pager
```

If another service owns port `514`, stop it or choose a different input port in `10-sdl.conf` and update sending devices.

### `/var/log/sdl-sender.log` is empty

Likely causes: rsyslog did not receive the message, the `omprog` action is not configured, or AppArmor blocked execution of `/usr/local/bin/sdl-sender.sh`.

```bash
sudo rsyslogd -N1
sudo journalctl -u rsyslog -n 80 --no-pager | grep -Ei 'omprog|apparmor|denied|permission|error|failed' || true
sudo aa-status 2>/dev/null | grep rsyslog || true
```

If AppArmor denial is present, apply the AppArmor include from step 5.

### Sender reports missing SDL settings

```bash
sudo stat -c '%U:%G:%a %n' /etc/sdl/sdl.env
sudo getent group syslog
sudo sh -c '. /etc/sdl/sdl.env; echo "url=$SDL_URL pass_length=${#SDL_PASS}"'
```

Expected:

```text
root:syslog:640 /etc/sdl/sdl.env
```

### HEC returns an error

Read the local sender log:

```bash
sudo tail -n 50 /var/log/sdl-sender.log
```

Common causes: wrong ingest hostname or region, wrong sourcetype path/query string, expired or incorrect write key, outbound firewall/proxy blocking HTTPS, or DNS resolution failure.

### SDL query returns zero matches

- Wait 10-30 seconds and retry.
- Confirm the query hostname is the SDL/XDR query host, not the ingest host.
- Confirm the read/query key is used in the JSON body as `token`.
- Query the exact `RUN_ID` printed during testing.
- Check `/var/log/sdl-sender.log` for HEC success responses.

### Expected warning in unprivileged containers

> [!NOTE]
> If this is tested in an unprivileged LXC/container, rsyslog may log:
> ```text
> imklog: cannot open kernel log (/proc/kmsg): Permission denied
> ```
> This warning is expected in many containers and does not prevent UDP/TCP syslog ingestion or SDL forwarding.

## Optional hardening after validation

After the pipeline is proven, consider removing the local raw `EVENT` copy from `/usr/local/bin/sdl-sender.sh` if local copies of forwarded events are not desired. Keep the HEC response logging or replace it with rate-limited operational logging according to your policy.

> [!TIP]
> You can also rotate `/var/log/sdl-sender.log` with logrotate if the host will forward sustained traffic:
>
> ```text
> /var/log/sdl-sender.log {
>     daily
>     rotate 7
>     compress
>     missingok
>     notifempty
>     copytruncate
> }
> ```
