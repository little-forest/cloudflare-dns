# cloudflare-dns-update

Keep multiple Cloudflare DNS A records in sync with the current global IP address.

* `cloudflare-dns` : calls the Cloudflare API (get / update / sync records)
* `dns-update` : runs `cloudflare-dns sync` and sends the result to Slack (used by systemd timers)

Requirements: `bash`, `curl`, `jq`

## Configuration file

Both commands read the same file. The first one found in the following order is loaded.

1. The file specified with `-c`
2. `./cloudflare-dns.conf`
3. `${HOME}/.cloudflare-dns.conf`
4. `/etc/cloudflare-dns.conf`

```bash
CLOUDFLARE_API_TOKEN=<YOUR_API_TOKEN>
ZONE_ID=<YOUR_ZONE_ID>
IP_COMMAND=(curl -s https://ifconfig.me)

# NAME[:TYPE[:TTL[:PROXIED]]]
#   TYPE    : A (default: A)
#   TTL     : 1-86400 or "auto" (default: keep current value)
#   PROXIED : true | false (default: keep current value)
RECORDS=(
  "example.com"
  "www.example.com:A:300"
  "vpn.example.com:A::false"
)

SLACK_WEBHOOK_URL=<YOUR_WEBHOOK_URL>
SLACK_CHANNEL="#<YOUR_CHANNEL>"
```

| Variable | Required | Used by | Description |
| --- | --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | yes | cloudflare-dns | API token with Zone.DNS:Edit permission |
| `ZONE_ID` | yes | cloudflare-dns | Can be overridden with `-z` |
| `IP_COMMAND` | for `sync` / `show-ip` | cloudflare-dns | Command (**array**) which prints the current IPv4 address in one line |
| `RECORDS` | unless `-n` is given | cloudflare-dns | Array of `NAME[:TYPE[:TTL[:PROXIED]]]` |
| `SLACK_WEBHOOK_URL` | | dns-update | No notification if empty |
| `SLACK_CHANNEL` / `SLACK_USER` / `SLACK_ICON` | | dns-update | `SLACK_USER` defaults to `hostname`, `SLACK_ICON` to `:ghost:` |

### Record spec `NAME[:TYPE[:TTL[:PROXIED]]]`

| Field | Value | When empty or omitted |
| --- | --- | --- |
| NAME | FQDN. A trailing `.` is removed and it is lowercased | required |
| TYPE | `A` only | `A` |
| TTL | Integer `1`-`86400`, or `auto` (= `1`) | keep the current value |
| PROXIED | `true` / `false` | keep the current value |

Records are never created. A record which does not exist in the zone is reported as `failed`.
All specs are validated before calling the API. If any of them is invalid (unknown type, bad TTL, too many fields, duplicated `NAME:TYPE`), nothing is done and the exit status is 2.

## cloudflare-dns

```
cloudflare-dns <get|update|sync|show-ip> [options]
  -c CONF   Config file path                          (--config=CONF)
  -z ZONE   Cloudflare zone ID (overrides ZONE_ID)
  -n SPEC   Target record: NAME[:TYPE[:TTL[:PROXIED]]]  (repeatable)
  -v VALUE  Value to set (update only)
  -a        All records in the zone (get only)
  -N        Dry run (update/sync)                     (--dry-run)
  -o FMT    text (default) | json  (update/sync)      (--output=FMT)
  -V        Verbose
  -h        Show usage                                (--help)
```

Without `-n`, all records in `RECORDS` are targeted. `-n` also accepts records which are not in `RECORDS`.

| Subcommand | Description |
| --- | --- |
| `get` | Print the target records as a JSON array. `-a` prints all records in the zone |
| `update -v VALUE` | Set VALUE to the target records (and TTL/PROXIED if specified) |
| `sync` | Same as `update`, using the IP address from `IP_COMMAND` (acquired once per run) |
| `show-ip` | Print the current IP address from `IP_COMMAND` (with retry) |

Each record results in `unchanged`, `updated`, `planned` (dry run) or `failed`. Records with no difference are not patched.
Even if some records fail, the remaining ones are processed.

```
$ cloudflare-dns sync
Current IP: 203.0.113.5
  updated    example.com          A  198.51.100.1 -> 203.0.113.5
  unchanged  www.example.com      A  203.0.113.5
  failed     missing.example.com  A  record not found
1 updated, 1 unchanged, 1 failed
```

With `-o json`:

```json
{
  "value": "203.0.113.5",
  "dry_run": false,
  "results": [
    {"name": "example.com", "type": "A", "status": "updated",
     "before": {"content": "198.51.100.1", "ttl": 1, "proxied": true},
     "after":  {"content": "203.0.113.5",  "ttl": 1, "proxied": true},
     "error": null},
    {"name": "missing.example.com", "type": "A", "status": "failed",
     "before": null, "after": null, "error": "record not found"}
  ]
}
```

| Exit status | Meaning |
| --- | --- |
| 0 | All records are `unchanged` / `updated` / `planned` |
| 1 | Some records are `failed`, or the IP address could not be acquired |
| 2 | Usage or configuration error. No API is called |

`CF_API_BASE` environment variable changes the API endpoint (default: `https://api.cloudflare.com/client/v4`). It is intended for testing with a local stub.

### Examples

* Show the records in `RECORDS`: `cloudflare-dns get`
* Show all records in the zone: `cloudflare-dns get -a`
* Check what will change: `cloudflare-dns sync -N`
* Update all records in `RECORDS`: `cloudflare-dns sync`
* Update a record which is not in `RECORDS`: `cloudflare-dns update -n other.example.com:A:auto -v 203.0.113.5`

## dns-update

```
dns-update [-c CONF] [-u] [-s] [-h]
  -c : config file path (passed through to cloudflare-dns)
  -u : update records (without -u: check only = cloudflare-dns sync -N)
  -s : silent on Slack unless something changed or failed
```

One Slack message is sent per run.

| Situation | Slack | Exit status |
| --- | --- | --- |
| All `unchanged` | Not sent with `-s`. Otherwise the current IP address | 0 |
| Some `planned` (without `-u`) | `<!channel>`, listed with 🔜, `(not updated)` in the headline | 0 |
| Some `updated`, none `failed` | `<!channel>`, listed with ✅ | 0 |
| Some `failed` | `<!channel>`, listed with ❌ and the reason | 1 |
| Fatal error | `<!channel> dns-update failed!` with the last 20 lines of stderr (sent even with `-s`) | non-zero |

```
<!channel> IP address has been changed!
Current IP : `203.0.113.5`
✅ `example.com` : `198.51.100.1` → `203.0.113.5`
➖ `www.example.com` : `203.0.113.5`
```

Fatal errors are: `cloudflare-dns` exits without a result (IP address acquisition failure, configuration error, legacy variables, ...), or `cloudflare-dns` is not found.
Errors before reading `SLACK_*` (e.g. the file given with `-c` does not exist) are printed only to stderr.

## Installing systemd unit files

```
sudo mkdir -p /opt/cloudflare-dns
sudo cp cloudflare-dns dns-update /opt/cloudflare-dns/
sudo cp conf-example/cloudflare-dns.conf /etc/cloudflare-dns.conf   # then edit it
sudo chmod 600 /etc/cloudflare-dns.conf
sudo cp systemd/cloudflare-dns*.service systemd/cloudflare-dns*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflare-dns.timer          # update records every 5 minutes
sudo systemctl enable --now cloudflare-dns-notify.timer   # notify the current IP address every hour
```

The current directory is `/` under systemd, so use `/etc/cloudflare-dns.conf` (the services pass it with `-c`).

## Migration from the single record version

The old format is rejected with exit status 2 and a migration message.

| Old | New |
| --- | --- |
| `dns-update.conf` | Move `SLACK_*` into `cloudflare-dns.conf` |
| `DOMAIN=example.com` | `RECORDS=("example.com")` |
| `IP_SUPPLIER_SCRIPT` / `IP_PROVIDER_SCRIPT` | `IP_COMMAND=(...)` (array) |
| `cloudflare-dns auto-update` | `cloudflare-dns sync` |
| `-t A -n NAME -l TTL` | `-n NAME:A:TTL` |
| `get` without arguments (all records in the zone) | `get -a` |
| `dns-update -d` | Write it in `RECORDS` |
| systemd `-c /etc/dns-update.conf` | `-c /etc/cloudflare-dns.conf` |
