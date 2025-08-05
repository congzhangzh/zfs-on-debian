# work with fast mirror

```bash
export DEBIAN_VERSION=bookworm
export DEB_PACKAGES_REPO=https://mirrors.163.com/debian/
export DEB_SECURITY_REPO=https://mirrors.163.com/debian-security/
```

# local debug

## improve the code somewhere
```bash
python -m http.server
```

## test on your vm or vps
1. boot to live system or rescue system
<!-- 2. prepare(needed?)
```bash
tee /etc/sysctl.d/10-bbr.conf <<EOF
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
EOF
sysctl --system
``` -->
2. debug new script
```bash
DEBUG=1
curl -sSL http://your-server:8000/hetzner-debian12-zfs-setup.sh | bash
```

# tips

you may need touch ~/.ssh/authorized_keys by yourself