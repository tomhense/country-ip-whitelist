# Country IP Whitelist

This is a program that fetches one or more country ip lists in cidr format and then creates a nftables config to whitelist these ip sets.


By doing this block all inbound traffic from countries other than the ones you whitelisted. It is to note that if you create a outbound connection to a server that is in a blocked country, then this connection will not be blocked (this is a feature and not a bug).


The generated nftables config should be imported in the main `/etc/nftables.conf` file, so you can still have other rules like only allowing access to a few ports.


This script uses the [lists from this project](https://github.com/ebrasha/cidr-ip-ranges-by-country/tree/master/CIDR).


## Example setup

### `/etc/nftables.conf`
```nftables
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;
        policy drop;

        # Always allow loopback and established connections
        iifname lo accept
        ct state established,related accept

        # Allow essential ports
        tcp dport { 80, 443, 22 } accept
    }
}

# Include dynamic whitelist.
include "/etc/nftables.d/whitelist.nft"
```

### `/etc/systemd/system/update-country-whitelist.service`
```ini
[Unit]
Description=Update nftables country whitelist
Documentation=man:systemd.service(5)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/update-country-whitelist
User=root
Group=root
Nice=10
StandardOutput=journal
StandardError=inherit

[Install]
WantedBy=multi-user.target
```

### `/etc/systemd/system/update-country-whitelist.timer`
```ini
[Unit]
Description=Run nftables whitelist updater weekly
Documentation=man:systemd.timer(5)

[Timer]
OnCalendar=Sun *-*-* 03:00:00
Persistent=true
RandomizedDelaySec=15min

[Install]
WantedBy=timers.target

```

## Setup
1. Customize the `update-country-whitelist` script and move it to `/usr/sbin`
2. Take the config files from the example section, customize them to your content and move them to the right places
3. Create the dir `/etc/nftables.d` if it does not exist
4. Reload systemd units: `sudo systemctl daemon-reload`
5. You may have to disable iptables or ufw if you use them **(CAUTION!!!)**
6. Enable nftables: `sudo systemctl enable nftables`
7. Enable the timer service: `sudo systemctl enable --now update-country-whitelist.timer`
8. Manually start the script once: `sudo systemctl start update-country-whitelist.timer`
9. **Test your firewall!!!**



