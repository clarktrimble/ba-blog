
# debian 11

## setup

### keep time

```bash
sudo apt purge ntp
sudo apt install systemd-timesyncd
```

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
