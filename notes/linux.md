
# debian 11

## setup

### keep time

```bash
sudo apt purge ntp
sudo apt install systemd-timesyncd
```

^^^ did not take, try againx - first reboot

```
sudo apt update
sudo apt upgrade
sudo reboot

sudo apt install systemd-timesyncd
sudo systemctl status systemd-timesyncd
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; preset: enabled)
     Active: active (running) since Sun 2023-12-31 08:42:41 CST; 18min ago
       Docs: man:systemd-timesyncd.service(8)
   Main PID: 891 (systemd-timesyn)
     Status: "Contacted time server 198.60.22.240:123 (0.debian.pool.ntp.org)."
      Tasks: 2 (limit: 18873)
     Memory: 1.3M
        CPU: 47ms
     CGroup: /system.slice/systemd-timesyncd.service
             └─891 /lib/systemd/systemd-timesyncd

Dec 31 08:42:41 tartu systemd[1]: Starting systemd-timesyncd.service - Network Time Synchronization...
Dec 31 08:42:41 tartu systemd[1]: Started systemd-timesyncd.service - Network Time Synchronization.
Dec 31 08:43:34 tartu systemd-timesyncd[891]: Contacted time server 198.60.22.240:123 (0.debian.pool.ntp.org).
Dec 31 08:43:34 tartu systemd-timesyncd[891]: Initial clock synchronization to Sun 2023-12-31 08:43:34.874148 CST.
```

looks promising!

or just for logses:

```
sudo journalctl -u systemd-timesyncd -b
Dec 31 08:42:41 tartu systemd[1]: Starting systemd-timesyncd.service - Network Time Synchronization...
Dec 31 08:42:41 tartu systemd[1]: Started systemd-timesyncd.service - Network Time Synchronization.
Dec 31 08:43:34 tartu systemd-timesyncd[891]: Contacted time server 198.60.22.240:123 (0.debian.pool.ntp.org).
Dec 31 08:43:34 tartu systemd-timesyncd[891]: Initial clock synchronization to Sun 2023-12-31 08:43:34.874148 CST.
```

And indeedy, times looks to be set :)


## commands

### show open ports

```bash
$ ss -tulnp
Netid           State            Recv-Q           Send-Q                      Local Address:Port                        Peer Address:Port           Process
udp             UNCONN           0                0                                 0.0.0.0:68                               0.0.0.0:*
tcp             LISTEN           0                4096                              0.0.0.0:8080                             0.0.0.0:*
tcp             LISTEN           0                4096                            127.0.0.1:41201                            0.0.0.0:*
tcp             LISTEN           0                128                               0.0.0.0:22                               0.0.0.0:*
tcp             LISTEN           0                4096                              0.0.0.0:9080                             0.0.0.0:*
tcp             LISTEN           0                128                                  [::]:22                                  [::]:*
```
