# cloudflare-dns-update

## Usage

```
  cloudflare-dns (get|update|auto-update|show-ip) [-h] [-z ZONE_ID] [-n NAME] [-t TYPE] [-l TTL] [-v VALUE]
    get:         Get DNS record (type and name should be specified)
    update:      Update DNS A record
    auto-update: Update DNS A record using IP_SUPPLIER_SCRIPT
    show-ip:     Show IP address using IP_SUPPLIER_SCRIPT
          -z : Cloudflare zone ID
          -n : DNS record name (ex: example.com)
          -t : DNS record type (ex: A, CNAME, TXT, ...)
          -v : DNS record value
            -l : TTL to set (optional)
          -V : Verbode mode
          -h : display usage
```

## Configuration file

You can configure some options in `${HOME}/.cloudflare-dns.conf` `/etc/cloudflare-dns.conf` `./cloudflare-dns.conf`.
Here is example configuration.

```
CLOUDFLARE_API_TOKEN=XXXXXXXXXXXXXXXXXXXX
ZONE_ID=XXXXXXXXXXXXXXXXXXXX
IP_SUPPLIER_SCRIPT=\"/usr/local/bin/xxxx\"
```

If both files are specified, `./cloudflare-dns.conf` will take precedence.

## Example

* Get all record set of specified zone.
  `cloudflare-dns get -z ZONE_ID -V`

* Get Sepcified DNS record.
  `cloudflare-dns get -z ZONE_ID -t A -n example.com`

* Update DNS A record.
  `cloudflare-dns update -z ZONE_ID -t A -n example.com -v IP_ADDRESS`

* Auto-Update DNS A record.
  (IP address is auto-accuired using IP_SUPPLIER_SCRIPT specified in cloudflare-dns.conf)
  `cloudflare-dns auto-update -z ZONE_ID -t A -n example.com`

## Installing systemd unit files

```
sudo cp systemd/cloudflare-dns.* /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable cloudflare-dns.timer
sudo systemctl start cloudflare-dns.timer
```

