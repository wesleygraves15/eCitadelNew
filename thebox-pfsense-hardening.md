# thebox - pfSense Firewall Hardening (172.21.0.254 LAN / .1.2/30 WAN)

**Why there is no `master-thebox.sh`:** pfSense is FreeBSD, not Linux. None of the
repo's tooling applies - no `apt`/`dnf`, no `iptables` (pfSense uses `pf`), no
systemd, no `bash` by default. Its entire state lives in `/cf/conf/config.xml`
and it is driven from the WebGUI / console, not from a hardening shell script.
Pushing the Linux scripts at it would do nothing useful and could brick console
access. So this box gets a checklist, not a script.

It is also the most important box you own: it is the edge. Harden it **first**,
back up its config **immediately**, and re-back-up after every change.

---

## 0. First 5 minutes (do these before anything else)

1. **Back up the config.** Diagnostics > Backup & Restore > Download. Do this
   *before* you touch anything, so you have the as-delivered state. Re-download
   after each milestone. (CLI: `cp /cf/conf/config.xml /root/config.baseline.xml`)
2. **Change the admin password.** Console option **3**, or System > User Manager.
   Assume `admin/pfsense` is known to red team.
3. **Inventory what's already there** - red team may have pre-seeded the config:
   - System > User Manager - extra users, SSH keys, API keys
   - Firewall > Rules (WAN **and** LAN) - any allow-any, any odd ports
   - Firewall > NAT - unexpected port-forwards into the scored hosts
   - System > Advanced > Admin Access - is the WebGUI/SSH exposed on WAN?
   - Services > Cron / Diagnostics > Command Prompt - `crontab -l`, weird jobs
   - System > Package Manager - anything you didn't install
   - System > Cert Manager - rogue certs
   - VPN (OpenVPN/IPsec/WireGuard) - any tunnel you didn't create

---

## 1. Management plane lockdown

- **WebGUI on LAN only**, HTTPS, non-default port. System > Advanced > Admin Access:
  - Protocol HTTPS, set a custom TCP port (e.g. 8443)
  - Leave **Anti-lockout** enabled until your explicit LAN rule is in and verified
  - Session timeout ~20 min; enable HSTS
- **No management on WAN.** Ensure there is no WAN rule allowing the GUI/SSH port.
- **SSH**: disable it (System > Advanced > Secure Shell) unless you actively need
  it. If enabled - key-only auth, LAN source only, and add an explicit firewall
  rule restricting the source to your team workstation.
- Separate admin accounts per operator; don't share the default `admin`.

---

## 2. Firewall ruleset (the actual job)

Default posture: **deny inbound on WAN**, allow only what's scored, filter egress
on LAN so a compromised internal host can't beacon freely.

- **WAN inbound**: default deny (pfSense default). Add port-forwards / rules ONLY
  for scored services to the specific internal host:
  - HTTP/HTTPS (80/443) -> concierge **172.21.0.102**
  - DB only if the DB is scored from outside the LAN -> blacklist **172.21.0.101**
    on 3306 or 5432 (most setups score the DB *through* the web app - don't expose
    it on WAN unless the scoreboard says so)
  - AD/DNS/Kerberos as required -> cabal **172.21.0.103**
- **WAN**: on a private `/30` lab uplink you will likely need to **uncheck
  "Block private networks"** on the WAN interface or the scoring traffic is dropped.
  Verify against your injects.
- **LAN egress filtering**: replace the default LAN "allow all to any" with an
  allow-list - DNS to your resolver/DC, HTTP/HTTPS for updates, NTP, and the
  agent ports your SOC uses (Salt 4505-4506, Wazuh 1514-1515, Splunk 9997). Then
  default-deny + log. This is what catches reverse shells.
- Enable **logging** on the default deny rules so drops show up in the SIEM.

---

## 3. Services & telemetry

- Turn off services you don't use: UPnP, DHCP on interfaces that don't need it,
  the DNS Resolver's outbound if not needed, NTP server on WAN, etc.
- **Remote syslog** to your aggregator: Status > System Logs > Settings > Remote
  Logging -> point at the Splunk/Wazuh box, ship firewall + system logs. The edge
  log is your highest-value detection feed.
- If packages are allowed: **Suricata** on the WAN (IDS), and pfBlockerNG. Tune
  before enabling block mode so you don't drop scored traffic.

---

## 4. Useful console / shell items (option 8)

```sh
# Show the running pf ruleset and live states
pfctl -sr
pfctl -ss

# Inspect the full config for anything you didn't put there
less /cf/conf/config.xml

# Snapshot the config off-box (do this repeatedly)
cp /cf/conf/config.xml /root/config.$(date +%s).xml

# Look for persistence
crontab -l
cat /etc/crontab
ls -la /usr/local/etc/rc.d/        # startup scripts

# Edit config safely, then reload everything
viconfig
/etc/rc.reload_all
```

---

## 5. Verify before you walk away

- From the WAN side, the scored services answer (curl the web box, etc.).
- From a LAN host, your egress allow-list works and a test to a blocked port is
  denied **and logged**.
- WebGUI/SSH are unreachable from WAN.
- A fresh config backup is saved off-box.
