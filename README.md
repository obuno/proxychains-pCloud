# pCloud through ProxyChains

NOTES:

Aloha, I've cobbled this because in my setups/personal environments, the only possible inernet connectivity is through proxy server(s) for both, HTTP(s) & external DNS resolutions. Natively, the pCloud Linux Client (nor any of their clients AFAIK) dose NOT support proxied connectivity, which is a shame if you ask me. In the hope that this might help some of you !!

## pCloud Linux Client:

https://www.pcloud.com/release-notes/linux.html   
You shall have pCloud Linux Client installed already, here is what I've done on my local distribution:
```
http_proxy=$http_proxy https_proxy=$http_proxy pamac install pcloud-drive
```
hence, for me this package: https://aur.archlinux.org/packages/pcloud-drive   
Pay attention NOT to allow/remove pCloud from your autostarted applications

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
sudo touch /etc/systemd/system/proxychains.dns.service
```
```
sudo bash -c 'cat <<EOF > /etc/systemd/system/proxychains.dns.service
[Unit]
Description=ProxyChains DNS Daemon
Requires=network.target
After=network.target

[Service]
Type=simple
User=%USERNAME%
ExecStart=/usr/bin/proxychains4-daemon
Restart=on-failure

StandardError=append:/var/log/proxychains/proxychains.dns.service.log

[Install]
WantedBy=multi-user.target
EOF'
```
NOTE: by default proxychains4-daemon listenip is 127.0.0.1, port 1053 and remotesubnet 224.

### ProxyChains pCloud service:
```
sudo touch /usr/bin/proxychains-pcloud.sh
```
```
sudo bash -c 'cat <<EOF > /usr/bin/proxychains-pcloud.sh
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
sudo chmod +x /usr/bin/proxychains-pcloud.sh
sudo chown %USERNAME%:%USERNAME% /usr/bin/proxychains-pcloud.sh
```
```
sudo bash -c 'cat <<EOF > /etc/systemd/system/proxychains.pcloud.service
[Unit]
Description=ProxyChains running pCloud
Requires=network.target
After=network.target

[Service]  
Type=simple
User=%USERNAME%
ExecStart=/usr/bin/proxychains-pcloud.sh
Restart=on-failure  

StandardError=append:/var/log/proxychains/proxychains.pcloud.service.log

[Install]
WantedBy=multi-user.target
EOF'
```

### ProxyChains logs folder:
```
sudo mkdir -p /var/log/proxychains/
```
```
------------------------------------------------------
Edit your ProxyChains configuration file:-------------
------------------------------------------------------

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
sudo systemctl daemon-reload

sudo systemctl enable proxychains.dns.service 
sudo systemctl enable proxychains.pcloud.service 

sudo systemctl start proxychains.dns.service
sudo systemctl start proxychains.pcloud.service
```

### Troubleshooting, Services Status & logs checkups:
```
sudo systemctl status proxychains.dns.service 
sudo systemctl status proxychains.pcloud.service 

tail -f /var/log/proxychains/proxychains.dns.service.log
tail -f /var/log/proxychains/proxychains.pcloud.service.log

```
