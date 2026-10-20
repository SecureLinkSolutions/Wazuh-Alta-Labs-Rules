# Wazuh Rules for Alta Labs Networking Gear

Custom Wazuh detection rules for syslog sent by Alta Labs devices (Route10 router, Alta access points and switches).

> This project is not affiliated with or endorsed by Alta Labs or Wazuh.

## What this does

Alta Control can forward device syslog to a remote host. Out of the box, Wazuh receives that syslog but raises no alerts, because none of the default rules match Alta's message formats. This rule file turns the useful messages into Wazuh alerts so they show up in Threat Hunting.

## What is covered

| Rule ID | Level | Source | Detects |
|---------|-------|--------|---------|
| 100201 | 3 | AP (hostapd) | Wi-Fi client connected |
| 100202 | 3 | AP (hostapd) | Wi-Fi client disconnected |
| 100203 | 8 | AP (hostapd) | Possible wrong Wi-Fi password attempt |
| 100204 | 5 | AP (hostapd) | Wi-Fi handshake timeout or wrong key |
| 100205 | 3 | Router (dnsmasq-dhcp) | Device requesting a DHCP lease |
| 100206 | 4 | Any device | Config pushed from Alta Control |
| 100207 | 3 | AP (wpa_supplicant) | Wireless mesh uplink connected |
| 100208 | 6 | AP (wpa_supplicant) | Wireless mesh uplink lost |
| 100209 | 6 | AP (wpa_supplicant) | Mesh uplink association rejected |
| 100210 | 3 | AP (hostapd) | Radio switched channel |
| 100211 | 4 | AP (hostapd) | Client dropped because Wi-Fi settings changed |
| 100212 | 0 | Any device (netd) | Single connectivity check failure (silent, feeds 100213) |
| 100213 | 8 | Any device (netd) | 10 connectivity failures in 60 seconds |
| 100214 | 5 | Any device | Lost connection to Alta Control |
| 100215 | 5 | Any device | Cannot resolve the Alta Control hostname |
| 100216 | 4 | AP (netd) | Using wireless mesh backhaul |
| 100217 | 3 | Any device (netd) | Using wired uplink |
| 100218 | 4 | Router (tailscaled) | Tailscale peer reached a LAN admin port (SSH, RDP, SMB or VNC) |

Rule IDs 100201 to 100230 are reserved for this project. If you already use IDs in that range, renumber before loading, because Wazuh silently ignores the second copy of a duplicated ID.

## Requirements

- Wazuh manager 4.x
- One or more Alta Labs devices managed in Alta Control
- Network path from the devices to the Wazuh manager on UDP 514

## Setup

### 1. Enable a syslog listener on the Wazuh manager

Add this block to `/var/ossec/etc/ossec.conf` inside `<ossec_config>`, next to your existing agent `<remote>` block. Do not replace the agent block.

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.X.X/24</allowed-ips>
</remote>
```

Change `allowed-ips` to your own LAN range, or to the exact IPs of your Alta devices. Restart the manager afterwards:

```
systemctl restart wazuh-manager
```

If the manager host has a firewall, allow UDP 514 from your Alta devices.

### 2. Point Alta Control at Wazuh

In Alta Control, open your site's **Settings**, expand **Advanced**, and set **Syslog Host** to:

```
<wazuh-ip>:514
```

Saving pushes the setting to every device. Devices may restart services or drop clients briefly while they apply it, so do it at a quiet time.

### 3. Install the rules

Copy `alta_rules.xml` to `/var/ossec/etc/rules/` on the manager, or add it in the dashboard under **Server management > Rules > Manage rules files**. Then restart the manager.

### 4. Test

Run `/var/ossec/bin/wazuh-logtest` and paste:

```
Oct 19 21:39:15 hostapd: 5h_2W47W98: AP-STA-CONNECTED d2:de:3f:aa:aa:aa
```

The output should end with rule 100201 and `**Alert to be generated.`

Then reconnect a phone to an Alta AP and search Threat Hunting for:

```
rule.groups: alta
```

Syslog events are attributed to the manager, so use the `location` field to see which Alta device sent each one.

## Notes and limitations

- **Only messages that Alta devices actually forward can be detected.** In testing, the Route10 forwarded very little beyond `tailscaled`, `htpdate` and `dnsmasq-dhcp`. It did not forward firewall hits, WAN failover events, banIP drops, SSH logins or IDS alerts. Those need a rule only if your firmware sends them.
- **Config replay lines are deliberately ignored.** On every config push the devices log their full config as `cfg: run "uci set ..."` lines. They are noisy and may contain secrets, so no rule matches them. Avoid keeping `logall` enabled, because it stores those lines in `archives.log`.
- **Rules 100212 and 100215 depend on Wazuh rule 1002.** Those messages contain the words "failure" or "failed", which the built-in rule 1002 already catches, so these rules are attached under it.
- **Connect and disconnect alerts can be frequent** on a busy network. If they are too noisy, lower rules 100201 and 100202 to level 2 so they stay out of alerts.
- **Verification status.** Rules 100201 and 100202 were confirmed against live events. The remaining rules were checked against real log lines captured from Alta devices, but have not all fired live. Rule 100203 uses a standard hostapd message that has not yet appeared in captured logs.

## Contributing

Pull requests and issues are welcome, especially:

- Sample log lines from other Alta models or firmware versions
- Rules for events your devices forward that are not covered here

Before posting sample logs, remove or replace:

- MAC addresses
- Public IP addresses
- Site IDs and DDNS hostnames (for example the string in `*.ddns.manage.alta.inc` names)
- Any tokens, keys or passwords

## License

MIT
