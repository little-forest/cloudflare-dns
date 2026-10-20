# cloudflare-dns

Keep multiple Cloudflare DNS A records in sync with the current global IP address.

`cloudflare-dns` calls the Cloudflare API (get / update / sync records) and, with `--notify`, sends the result to Slack. The systemd timers run it directly.

Requirements: `bash`, `curl`, `jq`

## Configuration file

The first one found in the following order is loaded.

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

| Variable | Required | Description |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | yes | API token with Zone.DNS:Edit permission |
| `ZONE_ID` | yes | Can be overridden with `-z` |
| `IP_COMMAND` | for `sync` / `show-ip` | Command (**array**) which prints the current IPv4 address in one line |
| `RECORDS` | unless `-n` is given | Array of `NAME[:TYPE[:TTL[:PROXIED]]]` |
| `SLACK_WEBHOOK_URL` | for `--notify` | Slack incoming webhook URL |
| `SLACK_CHANNEL` / `SLACK_USER` / `SLACK_ICON` | | `SLACK_USER` defaults to `hostname`, `SLACK_ICON` to `:ghost:` |

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
  -o FMT    text (default) | json  (update/sync/show-ip)  (--output=FMT)
            json: one JSON object per record (show-ip: per IP address) on
                  stdout, and one JSON object per message on stderr
  --notify[=MODE]  Send the result to Slack (sync/show-ip)
            always (default): every run
            changes: only when something is updated, planned or failed
                     (sync only)
  -V        Verbose
  -h        Show usage                                (--help)
```

Without `-n`, all records in `RECORDS` are targeted. `-n` also accepts records which are not in `RECORDS`.

| Subcommand | Description |
| --- | --- |
| `get` | Print the target records as a JSON array. `-a` prints all records in the zone |
| `update -v VALUE` | Set VALUE to the target records (and TTL/PROXIED if specified) |
| `sync` | Same as `update`, using the IP address from `IP_COMMAND` (acquired once per run) |
| `show-ip` | Print the current IP address from `IP_COMMAND` (with retry). `-o json` prints it as a JSON object. `--notify` sends it to Slack |

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

With `-o json`, one JSON object is printed per record (one line each, compact, no color, no timestamp). The systemd services use this format so that the results in the journal can be sent to a log collector such as Datadog, or filtered with `jq`.

`sync --notify=changes`:

```json
{"level":"info","currentIP":"203.0.113.5","status":"updated","record":"example.com","type":"A","oldIP":"198.51.100.1","newIP":"203.0.113.5","oldDnsProxy":true,"newDnsProxy":true,"oldTtl":1,"newTtl":1}
{"level":"debug","currentIP":"203.0.113.5","status":"unchanged","record":"www.example.com","type":"A","oldIP":"203.0.113.5","newIP":"203.0.113.5","oldDnsProxy":false,"newDnsProxy":false,"oldTtl":300,"newTtl":300}
{"level":"error","currentIP":"203.0.113.5","status":"failed","record":"missing.example.com","type":"A","oldIP":null,"newIP":null,"oldDnsProxy":null,"newDnsProxy":null,"oldTtl":null,"newTtl":null,"error":"record not found"}
```

| Key | Type | Description |
| --- | --- | --- |
| `level` | string | `error` for `failed`, `debug` for `unchanged` with `--notify=changes`, otherwise `info` |
| `currentIP` | string | The IP address from `IP_COMMAND` (`sync`), or `-v VALUE` (`update`) |
| `status` | string | `updated` / `planned` / `unchanged` / `failed` |
| `record` | string | Record name (lowercased, without the trailing `.`) |
| `type` | string | `A` |
| `oldIP` / `newIP` | string \| null | `content` before / after the update (planned value for `planned`) |
| `oldDnsProxy` / `newDnsProxy` | boolean \| null | `proxied` before / after the update |
| `oldTtl` / `newTtl` | number \| null | `ttl` before / after the update (`1` is auto) |
| `error` | string | The reason of `failed`. Only present in `failed` records |

`level` is decided per record. Only `--notify=changes`, which does not send a message for unchanged records, lowers `unchanged` to `debug`.

| Result of the run | `--notify` (= `--notify=always`) | `--notify=changes` | Without `--notify` |
| --- | --- | --- | --- |
| All `unchanged` | Sent / each record `info` | Not sent / each record `debug` | Not sent / each record `info` |
| Some `updated` / `planned` | Sent / all records `info` | Sent / changed records `info`, `unchanged` records `debug` | Not sent / all records `info` |
| Some `failed` | Sent / the record `error` | Sent / the record `error` | Not sent / the record `error` |

All keys except `error` are always present. The `old*` and `new*` values are `null` when the record is not found, and the `new*` values are `null` when the update itself failed.

`show-ip -o json` prints one line instead of the IP address. `level` is always `info`, with or without `--notify`.

```json
{"level":"info","currentIP":"203.0.113.5"}
```

With `-o json`, stderr is also one JSON object per line (compact, no color), for all subcommands. `level` is `error`, `warning` (retry of `IP_COMMAND`) or `debug` (`-V`). The migration message for legacy variables is put in `hint`.

```json
{"level":"warning","message":"Invalid IP address, retry after 5sec... : "}
{"level":"error","message":"Couldn't get IP address. (Executed: curl -s https://ifconfig.me Result: )"}
{"level":"debug","message":"GET https://api.cloudflare.com/client/v4/zones/xxx/dns_records?..."}
```

Exceptions (printed as text):

* Errors before `-o json` is read, and an invalid `-o` value: `jq` is not found, invalid long options (`--notify=foo` etc.), invalid subcommands, and options before `-o`. They exit with status 2.
* The stderr of `IP_COMMAND` itself. Use a quiet command such as `IP_COMMAND=(curl -s https://ifconfig.me)` to keep the output JSON only.

The error of `curl` is included in the error message (e.g. `... (curl exit code: 6) (curl: (6) Could not resolve host: api.cloudflare.com)`) instead of being printed on its own line. This applies to all output formats.

```
journalctl -u cloudflare-dns -o cat | jq -c 'select(.level != "debug")'
```

| Exit status | Meaning |
| --- | --- |
| 0 | All records are `unchanged` / `updated` / `planned` |
| 1 | Some records are `failed`, or the IP address could not be acquired |
| 2 | Usage or configuration error. No API is called |

`CF_API_BASE` environment variable changes the API endpoint (default: `https://api.cloudflare.com/client/v4`). It is intended for testing with a local stub.

### Slack notification (`--notify`)

`sync --notify` and `show-ip --notify` send one Slack message per run. Without `--notify`, nothing is sent even if `SLACK_WEBHOOK_URL` is set.
The output to stdout and the exit status are the same as without `--notify`, except `level` of `sync -o json` (`--notify=changes` lowers `unchanged` to `debug`).

`sync --notify`:

| Situation | `--notify` (= `--notify=always`) | `--notify=changes` | Exit status |
| --- | --- | --- | --- |
| All `unchanged` | The current IP address | Not sent | 0 |
| Some `planned` (with `-N`) | `<!channel>`, listed with 🔜, `(not updated)` in the headline | same | 0 |
| Some `updated`, none `failed` | `<!channel>`, listed with ✅ | same | 0 |
| Some `failed` | `<!channel>`, listed with ❌ and the reason | same | 1 |
| Fatal error | `<!channel> cloudflare-dns failed!` with the error messages of the run (up to 20 lines) | same | 1 or 2 |

```
<!channel> IP address has been changed!
Current IP : `203.0.113.5`
✅ `example.com` : `198.51.100.1` → `203.0.113.5`
➖ `www.example.com` : `203.0.113.5`
```

`show-ip --notify` (`--notify=changes` is rejected with exit status 2):

| Situation | Message | Exit status |
| --- | --- | --- |
| The IP address is acquired | ``Current IP : `203.0.113.5` `` (the same text as `sync --notify` with all `unchanged`) | 0 |
| Fatal error | `<!channel> cloudflare-dns failed!` with the error messages of the run | 1 or 2 |

Fatal errors are: the IP address could not be acquired, or `ZONE_ID` / `CLOUDFLARE_API_TOKEN` / `RECORDS` / `IP_COMMAND` is invalid (`show-ip` checks only `IP_COMMAND`).
The following errors are printed only to stderr, because they occur before the Slack settings are validated: invalid options, the config file is not found, legacy variables, and `--notify` without `SLACK_WEBHOOK_URL` (exit status 2).

A failure to send the message to Slack (including HTTP errors) is printed to stderr. It does not change the exit status.

### Examples

* Show the records in `RECORDS`: `cloudflare-dns get`
* Show all records in the zone: `cloudflare-dns get -a`
* Check what will change: `cloudflare-dns sync -N`
* Update all records in `RECORDS`: `cloudflare-dns sync`
* Update all records and notify Slack only when something changed: `cloudflare-dns sync --notify=changes`
* Update a record which is not in `RECORDS`: `cloudflare-dns update -n other.example.com:A:auto -v 203.0.113.5`

## Installing systemd unit files

```
sudo mkdir -p /opt/cloudflare-dns
sudo cp cloudflare-dns /opt/cloudflare-dns/
sudo cp conf-example/cloudflare-dns.conf /etc/cloudflare-dns.conf   # then edit it
sudo chmod 600 /etc/cloudflare-dns.conf
sudo cp systemd/cloudflare-dns*.service systemd/cloudflare-dns*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflare-dns-sync.timer      # update records every 5 minutes (notify only on changes)
sudo systemctl enable --now cloudflare-dns-show-ip.timer   # notify the current IP address at 00:02, 06:02, 12:02 and 18:02
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
| `dns-update -s -u` | `cloudflare-dns sync --notify=changes` |
| `dns-update` (without `-u`) | `cloudflare-dns show-ip --notify` (the current IP address only), or `cloudflare-dns sync -N --notify` (with the differences from the records) |
| `/opt/cloudflare-dns/dns-update` | Not used any more. Remove it |
| systemd `dns-update -c /etc/dns-update.conf ...` | Replace the unit files with the ones in `systemd/` |
| systemd `cloudflare-dns.{service,timer}` | `cloudflare-dns-sync.{service,timer}` |
| systemd `cloudflare-dns-notify.{service,timer}` | `cloudflare-dns-show-ip.{service,timer}` |

If the old units are installed, remove them before installing the new ones. Otherwise both timers run.

```
sudo systemctl disable --now cloudflare-dns.timer cloudflare-dns-notify.timer
sudo rm /etc/systemd/system/cloudflare-dns.{service,timer} /etc/systemd/system/cloudflare-dns-notify.{service,timer}
sudo systemctl daemon-reload
```
