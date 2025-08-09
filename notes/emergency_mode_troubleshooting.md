# ZFS 系统紧急模式故障排除指南

## 问题现象
系统启动后只能进入紧急模式（Emergency Mode），无法正常启动到用户界面。

## 常见原因分析

### 1. ZFS Pools 无法导入
- Boot pool (bpool) 导入失败
- Root pool (rpool) 导入失败  
- Pool 状态异常或损坏

### 2. 加密 Root Pool 解锁失败
- 密码输入错误或超时
- Dropbear 远程解锁服务故障
- 加密密钥丢失

### 3. 内核模块问题
- ZFS 内核模块未加载
- 内核版本与 ZFS 模块不兼容
- initramfs 缺少必要模块

### 4. 文件系统挂载问题
- /boot 挂载失败
- 根文件系统挂载失败
- fstab 配置错误

## 紧急模式下的诊断步骤

### 第一步：检查 ZFS 服务状态

```bash
# 检查 ZFS 内核模块是否加载
lsmod | grep zfs

# 手动加载 ZFS 模块（如果未加载）
modprobe zfs

# 检查 ZFS 服务状态
systemctl status zfs-import-cache
systemctl status zfs-import.target
systemctl status zfs-mount
systemctl status zfs.target
```

### 第二步：检查 ZFS Pools 状态

```bash
# 查看可用的 pools
zpool import

# 查看当前导入的 pools
zpool list
zpool status

# 检查 pool 缓存
cat /etc/zfs/zpool.cache 2>/dev/null || echo "No cache file"

# 查看 ZFS 数据集
zfs list
```

### 第三步：尝试手动导入 Pools

```bash
# 导入 boot pool（通常名为 bpool）
zpool import -f bpool

# 导入 root pool（通常名为 rpool）
zpool import -f rpool

# 如果 pool 名称不确定，先查看可用的
zpool import | grep "pool:"
```

### 第四步：处理加密 Pool

如果 root pool 是加密的：

```bash
# 解锁加密的数据集
zfs load-key rpool/ROOT/debian
# 或者
zfs load-key -a  # 解锁所有加密数据集

# 检查加密状态
zfs get encryption,keystatus,keyformat rpool/ROOT/debian
```

### 第五步：检查挂载点

```bash
# 查看 ZFS 挂载状态
zfs get mounted,mountpoint

# 尝试手动挂载关键文件系统
zfs mount rpool/ROOT/debian  # 挂载根文件系统
zfs mount bpool/BOOT/debian  # 挂载 /boot

# 检查挂载结果
mount | grep zfs
df -h
```

### 第六步：修复系统服务

```bash
# 重新启动 ZFS 相关服务
systemctl restart zfs-import-cache
systemctl restart zfs-mount
systemctl restart zfs.target

# 检查服务日志
journalctl -u zfs-import-cache
journalctl -u zfs-mount
journalctl -u zfs.target
```

## 常见解决方案

### 解决方案 1：Pool 导入问题

如果 pools 无法自动导入：

```bash
# 强制导入 pools
zpool import -f -R / bpool
zpool import -f -R / rpool

# 设置正确的挂载点
zfs set mountpoint=/boot bpool/BOOT/debian
zfs set mountpoint=/ rpool/ROOT/debian

# 挂载文件系统
zfs mount bpool/BOOT/debian
zfs mount rpool/ROOT/debian
```

### 解决方案 2：重建 initramfs

如果怀疑 initramfs 问题：

```bash
# 挂载系统分区（如果还没挂载）
mount -t zfs rpool/ROOT/debian /mnt
mount -t zfs bpool/BOOT/debian /mnt/boot
mount --rbind /dev /mnt/dev
mount --rbind /proc /mnt/proc
mount --rbind /sys /mnt/sys

# 进入 chroot 环境
chroot /mnt /bin/bash

# 重建 initramfs
update-initramfs -u -k all

# 更新 GRUB
update-grub

# 退出 chroot 并重启
exit
umount -R /mnt
reboot
```

### 解决方案 3：修复 ZFS 缓存

```bash
# 删除旧缓存
rm -f /etc/zfs/zpool.cache

# 重新生成缓存
zpool set cachefile=/etc/zfs/zpool.cache bpool
zpool set cachefile=/etc/zfs/zpool.cache rpool

# 确保缓存文件权限正确
chmod 644 /etc/zfs/zpool.cache
```

### 解决方案 4：检查和修复 fstab

```bash
# 检查 fstab 配置
cat /etc/fstab

# 确保没有冲突的 ZFS 挂载条目
# ZFS 数据集不应该在 fstab 中重复定义
# 只保留 EFI 分区（如果是 EFI 系统）和其他非 ZFS 分区
```

## 键盘布局问题处理

如果在紧急模式下遇到键盘布局问题：

```bash
# 临时切换到美式键盘布局
loadkeys us

# 或者加载德语布局
loadkeys de

# 查看可用的键盘布局
ls /usr/share/keymaps/i386/qwerty/
ls /usr/share/keymaps/i386/qwertz/

# 测试特殊字符
echo 'Test: @ # $ % ^ & * ( ) [ ] { } | \'
```

## 预防措施

### 1. 定期备份 ZFS 配置

```bash
# 备份 pool 配置
zpool status > /root/zfs-status-backup.txt
zfs list > /root/zfs-list-backup.txt
cp /etc/zfs/zpool.cache /root/zpool.cache.backup
```

### 2. 创建救援脚本

创建一个自动修复脚本放在 /root/zfs-emergency-repair.sh：

```bash
#!/bin/bash
echo "ZFS Emergency Repair Script"
echo "Attempting to import pools..."

# 导入 pools
zpool import -f bpool 2>/dev/null
zpool import -f rpool 2>/dev/null

# 挂载关键文件系统
zfs mount rpool/ROOT/debian 2>/dev/null
zfs mount bpool/BOOT/debian 2>/dev/null

echo "Current ZFS status:"
zpool status
zfs list

echo "Mounted filesystems:"
mount | grep zfs
```

### 3. 监控 ZFS 健康状态

```bash
# 检查 pool 健康状态
zpool status -x

# 定期清理（scrub）
zpool scrub bpool
zpool scrub rpool

# 查看 scrub 进度
zpool status
```

## 最后手段：使用救援系统

如果所有方法都失败，可能需要：

1. **从安装 USB 启动进入救援模式**
2. **重新安装系统**（数据可能会丢失）
3. **联系技术支持**

## 日志收集

在寻求帮助时，收集以下信息：

```bash
# 系统启动日志
journalctl -b | grep -i zfs > /tmp/zfs-boot.log

# ZFS 服务日志  
journalctl -u zfs-import-cache > /tmp/zfs-import.log
journalctl -u zfs-mount > /tmp/zfs-mount.log

# 系统信息
uname -a > /tmp/system-info.txt
lsmod | grep zfs >> /tmp/system-info.txt
zpool status >> /tmp/system-info.txt
zfs list >> /tmp/system-info.txt
```

## 总结

ZFS 系统的紧急模式问题通常可以通过手动导入 pools 和挂载文件系统来解决。关键是要有耐心，按步骤诊断问题。大多数情况下，问题都可以在不丢失数据的情况下修复。
