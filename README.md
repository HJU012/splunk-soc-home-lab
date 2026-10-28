# Splunk SOC Home Lab

A home SIEM lab built to practise log collection, searching and basic detection with Splunk Enterprise. A Kali Linux VM forwards its authentication logs to a Splunk server running on Ubuntu, where failed logins can be searched for in real time.

## Architecture

```
Windows 11 host (Oracle VirtualBox, bridged networking)
|
+-- Ubuntu-Splunk-SOC VM : Splunk Enterprise 10.6.1.1 (indexer + web UI)
|       web UI: port 8000 | receiving port: 9997
|            ^
|            | logs (TCP 9997)
|            |
+-- KALI-LINUX VM : Splunk Universal Forwarder 10.6.1.1 (log source)
        monitors /var/log/auth.log
```

**Data flow:** Kali `auth.log` -> Universal Forwarder -> TCP 9997 -> Splunk Enterprise (`index=main`) -> Search & Reporting.

## Environment

| Component | Detail |
|---|---|
| Host | Windows 11 laptop, 8 GB RAM |
| Hypervisor | Oracle VirtualBox, bridged adapter |
| SIEM server | Ubuntu Server 26.04 LTS, 2 vCPU, 3 GB RAM, 60 GB disk |
| Log source | Kali Linux VM (hostname `NeonByteN`) |
| Splunk Enterprise | 10.6.1.1 |
| Splunk Universal Forwarder | 10.6.1.1 |
| Ports | 8000 (web UI), 9997 (receiving) |

## Setup summary

1. Created two VirtualBox VMs on a bridged network so the host, Ubuntu and Kali are on the same LAN.
2. Copied the Splunk Enterprise `.deb` package to Ubuntu with `scp` and installed it with `apt install ./splunk-*.deb`.
3. Created a dedicated `splunk` user, started Splunk as that user and created the admin account.
4. Enabled receiving on port 9997 (Settings > Forwarding and receiving > Configure receiving).
5. Installed the Universal Forwarder on Kali, created a `splunk` user and started the forwarder.
6. Pointed the forwarder at the Splunk server: `splunk add forward-server 192.168.1.5:9997`.
7. Added `/var/log/auth.log` as a monitored input into `index=main`.
8. Verified that Kali events arrive in Splunk.

## Configuration (Kali forwarder)

`/opt/splunkforwarder/etc/system/local/outputs.conf`

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.1.5:9997

[tcpout-server://192.168.1.5:9997]
```

`/opt/splunkforwarder/etc/apps/search/local/inputs.conf`

```ini
[monitor:///var/log/auth.log]
disabled = false
index = main
```

No credentials are stored in this repository.

## Detection use case: failed logins

```spl
index=main "authentication failure"
```

Running `sudo` on Kali with a wrong password writes a `pam_unix(sudo:auth): authentication failure` event to `auth.log`. The forwarder ships it to Splunk within seconds, where it can be searched, counted and turned into an alert. This is the basis of a brute-force login detection.

## Screenshots

Click a link to open the screenshot.

1. [Both VMs running in VirtualBox](01-virtualbox-vms.png.png)
2. [Splunk running on the Ubuntu server (IP and service status)](02-ubuntu-splunk-running.png.png)
3. [Splunk Enterprise web interface](03-splunk-web-home.png.png)
4. [Receiving port 9997 enabled](04-receiving-port-9997.png.png)
5. [Forwarder connected to the Splunk server (Active forwards)](05-kali-forwarder-connected.png.png)
6. [Forwarder configuration files](06-forwarder-configs.png.png)
7. [Kali events searchable in Splunk](07-splunk-search-events.png.png)
8. [Generating a failed sudo attempt on Kali](08-failed-sudo-attempt.png.png)
9. [The failed login detected in Splunk](09-detection-auth-failure.png.png)

## Troubleshooting log

| Problem | Cause | Fix |
|---|---|---|
| Splunk would not start as root | Splunk 10.x deprecates running as root | Created a dedicated `splunk` user, set ownership on `/opt/splunk` and started Splunk as that user |
| Forwarder error: "No users exist" | The first start failed before the admin account was created, leaving an empty `passwd` file | Created `user-seed.conf`, removed the empty `etc/passwd` and restarted |
| Forwarder stopped matching the server after a reboot | Ubuntu received a new DHCP address (192.168.1.6 to 192.168.1.5) | Removed and re-added the forward-server with the new IP |
| `apt` could not find the downloaded package | `apt` needs `./` to treat the file as a local package | `sudo apt install ./package.deb` |
| "Parameters must be in the form -parameter value" | Spaces around the colon in `-auth user:password` | Used `-auth user:password` with no spaces |

## Observations and limitations

- Splunk auto-assigned the sourcetype `auth-too_small` because the log file was tiny at first. The proper sourcetype is `linux_secure`.
- The Splunk web UI shows UTC while the raw log lines carry a +05:30 offset. The user time zone can be set to Asia/Kolkata.
- The VMs use DHCP, so addresses can change after a reboot.

## Next steps

- [ ] Add the Windows host as a second log source
- [ ] Create an alert for repeated failed logins
- [ ] Set the sourcetype to `linux_secure`
- [ ] Set the Splunk time zone to Asia/Kolkata
- [ ] Build a dashboard for authentication events
- [ ] Enable boot-start for Splunk and the forwarder

## Skills practised

Linux administration, VirtualBox networking, Splunk deployment, forwarder configuration, SPL searching, log analysis and troubleshooting.

---

*Lab environment for learning only. Author: Nandan A D*
