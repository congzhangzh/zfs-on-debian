# with Debian live CD on remote server or virtual machine
```bash
sudo apt update && sudo apt install htop screen openssh-server vim -y 
sudo passwd user
```

# local control client
```bash
#Tips: use you real ip here
ssh-copy-id user@your_ip
ssh user@your_ip
#TODO
[ -d ~root/.ssh ] || sudo mkdir -p ~root/.ssh
sudo cp ~/.ssh/authorized_keys ~root/.ssh
# for safe
screen -S zfs # your can recover by **screen -x zfs** if later reconnect
sudo su
#Tips: use -h to get what you can customization
#curl -sSl  https://raw.githubusercontent.com/congzhangzh/zfs-on-debian/main/debian-zfs-setup.sh | bash -s -- -h
curl -sSl https://raw.githubusercontent.com/congzhangzh/zfs-on-debian/main/debian-zfs-setup.sh | bash -s --  --debug --no-reboot --keyboard-layout us --ipv4-only
```
# Debug Tips
```bash
# touch ~/.ssh/authorized_keys
cat /tmp/zfs-hetzner-vm/disks.log
cat /tmp/zfs-hetzner-vm/install.log
```

