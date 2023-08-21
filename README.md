# pCloud Drive for Linux through ProxyChains

NOTES:

Aloha, I've cobbled this because in my setups/personal environments, the only possible internet connectivity is through proxy server(s) for both, HTTP(s) & external DNS resolutions. Natively, neither the pCloud Drive Client for Linux nor any of the pCloud proposed Drive Clients (AFAIK) seems to support proxied connectivity, which is somewhat of a shame. In the hope that this might help some of you !!

To be edited on your system:
- %USERNAME% | replace this value with your elected user account which will run the pCloud binaries.
- /usr/bin/pcloud | this is where the pCloud binary was installed on my system, this might diverge on your system.

## Installing the pCloud Drive Linux Client:

You shall have pCloud Linux Client installed already, here is what I've done on my local distribution:
```
http_proxy=$http_proxy https_proxy=$http_proxy pamac install pcloud-drive
```
Hence, for me this package: https://aur.archlinux.org/packages/pcloud-drive   
The pCloud Drive Client for Linux release notes are here: https://www.pcloud.com/release-notes/linux.html

Pay attention NOT to allow OR remove pCloud from your autostarted applications.

## Installing ProxyChains NG:

ProxyChains NG GitHub repo is available here: https://github.com/rofl0r/proxychains-ng

*** Installation *** (needs a working C compiler, preferably gcc)
```
git clone https://github.com/rofl0r/proxychains-ng
cd proxychains-ng

./configure --prefix=/usr --sysconfdir=/etc
make
sudo make install
sudo make install-config (installs proxychains.conf)
```

## Creating your systemd services:

### ProxyChains DNS daemon service:
```
sudo touch /etc/systemd/user/proxychains.dns.service
```
```
sudo bash -c 'cat <<EOF > /etc/systemd/user/proxychains.dns.service
[Unit]
Description=ProxyChains DNS Daemon

[Service]
Type=simple
ExecStart=/usr/bin/proxychains4-daemon
Restart=on-failure

StandardError=append:/var/log/proxychains/proxychains.dns.service.log

[Install]
WantedBy=default.target
EOF'
```
NOTE: by default proxychains4-daemon listenip is 127.0.0.1, port 1053 and remotesubnet 224.

### ProxyChains pCloud service:
```
sudo touch /usr/local/bin/proxychains-pcloud.sh
```
```
sudo bash -c 'cat <<EOF > /usr/local/bin/proxychains-pcloud.sh
#!/bin/bash

#########################
# umounting older pcloud-drive-x.x.AppImage mount points
#########################
/usr/bin/umount /tmp/.mount_pcloud*
#########################

sleep 10

#########################
# display settings may vary on your system, edit the line below accordingly
#########################
export DISPLAY=:1
#########################

/usr/bin/proxychains4 /usr/bin/pcloud
EOF'
```
```
sudo chmod +x /usr/local/bin/proxychains-pcloud.sh
```
```
sudo bash -c 'cat <<EOF > /etc/systemd/user/proxychains.pcloud.service
[Unit]
Description=ProxyChains running pCloud

[Service]
Type=simple
ExecStart=/usr/local/bin/proxychains-pcloud.sh
Restart=on-failure

StandardError=append:/var/log/proxychains/proxychains.pcloud.service.log

[Install]
WantedBy=default.target
EOF'
```

### Creating the ProxyChains logs folder:
```
sudo mkdir -p /var/log/proxychains/
```

### Let user write to the ProxyChains logs folder:
```
sudo setfacl -R -m u:%USERNAME%:rwX /var/log/proxychains/
```

### Editing your ProxyChains configuration file:
```
Default proxychains.conf file settings:
------------------------------------------------------
grep "^[^#;]" /tmp/proxychains-ng/src/proxychains.conf
------------------------------------------------------
strict_chain
proxy_dns
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
socks4  127.0.0.1 9050
```
```
Edited proxychains.conf file settings:
------------------------------------------------------
grep "^[^#;]" /etc/proxychains.conf
------------------------------------------------------
strict_chain
proxy_dns_daemon 127.0.0.1:1053
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
http  10.11.12.13  8080
```

### Enable & Start the created services:
```
systemctl --user daemon-reload

systemctl --user enable proxychains.dns.service
systemctl --user enable proxychains.pcloud.service 

systemctl --user start proxychains.dns.service
systemctl --user start proxychains.pcloud.service
```

### Troubleshooting, Services Status & logs checkups:
```
systemctl --user status proxychains.dns.service
systemctl --user status proxychains.pcloud.service

tail -f /var/log/proxychains/proxychains.dns.service.log
tail -f /var/log/proxychains/proxychains.pcloud.service.log

```
