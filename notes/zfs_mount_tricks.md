# ZFS挂载机制学习笔记

## 1. ZFS挂载相关参数详解

### 1.1 zpool级别参数 (`-m`)

```bash
# 控制存储池本身的挂载行为
zpool create -m <option> poolname disk

# 可选值：
-m none          # 存储池不会自动挂载
-m legacy        # 使用传统挂载方式（需要/etc/fstab）
-m /path         # 指定存储池根数据集的挂载点
```

**使用场景对比：**
- `none`: 系统池（如引导池），不需要默认挂载
- `legacy`: 需要与传统工具集成
- `路径`: 数据池，自动挂载到指定位置

### 1.2 数据集级别参数 (`-O`)

```bash
# 设置数据集的默认属性
zpool create -O mountpoint=<path> poolname disk

# 常用值：
-O mountpoint=none     # 数据集不会被挂载
-O mountpoint=/path    # 数据集挂载到指定路径
-O mountpoint=legacy   # 使用传统挂载方式
```

### 1.3 canmount属性

```bash
# 控制数据集的挂载行为
zfs create -o canmount=<option> dataset

# 可选值：
canmount=on       # 自动挂载（默认）
canmount=off      # 永不挂载（容器数据集）
canmount=noauto   # 手动挂载（根文件系统）
```

## 2. 参数优先级和协作关系

### 2.1 优先级顺序
```
数据集属性 > 池的默认属性 > 系统默认值
```

### 2.2 实际案例分析

```bash
# 创建池时的设置
zpool create \
  -m none \                    # 池级别：不自动挂载
  -O mountpoint=none \         # 数据集默认：挂载点为none
  -R /mnt \                   # 临时根目录
  rpool disk1

# 创建根文件系统数据集
zfs create \
  -o canmount=noauto \        # 覆盖默认：手动挂载
  -o mountpoint=/ \           # 覆盖默认：挂载点为根
  rpool/ROOT/debian

# 结果：数据集知道要挂载到/，但不会自动挂载
```

## 3. 系统启动过程中的挂载流程

### 3.1 GRUB阶段
```bash
# GRUB功能：
- 直接读取ZFS文件（无需挂载）
- 加载内核和initramfs
- 传递root=ZFS=rpool/ROOT/debian参数

# grub.cfg示例：
linux /vmlinuz root=ZFS=rpool/ROOT/debian
initrd /initrd.img
```

### 3.2 Initramfs阶段
```bash
# ZFS initramfs脚本执行顺序：
1. zpool import -N rpool        # 导入池但不挂载
2. zpool import -N bpool        # 导入引导池
3. zfs mount rpool/ROOT/debian  # 手动挂载根文件系统

# 此时挂载状态：
/ (根) ← rpool/ROOT/debian (已挂载)
/boot ← bpool/BOOT/debian (未挂载)
/boot/efi ← EFI分区 (未挂载)
```

### 3.3 Systemd接管阶段
```bash
# systemd根据fstab挂载其余文件系统：

# 1. Boot pool (使用legacy模式)
bpool/BOOT/debian /boot zfs \
  nodev,relatime,x-systemd.requires=zfs-mount.service 0 0

# 2. EFI分区 (传统FAT32文件系统)
/dev/disk/by-partuuid/xxx /boot/efi vfat defaults 0 1

# 3. Swap (ZFS volume)
/dev/zvol/rpool/swap none swap discard 0 0
```

## 4. 为什么根文件系统使用canmount=noauto？

### 4.1 时序控制
```bash
# 问题：如果使用canmount=on
zfs create -o mountpoint=/ rpool/ROOT/debian
# 结果：立即尝试挂载到/，与当前系统冲突

# 解决：使用canmount=noauto
zfs create -o canmount=noauto -o mountpoint=/ rpool/ROOT/debian
# 结果：数据集存在但不自动挂载，由initramfs精确控制
```

### 4.2 避免重复挂载
```bash
# 启动流程中的挂载责任分工：
- initramfs: 负责挂载根文件系统
- systemd: 负责挂载其他文件系统
- ZFS自动挂载: 被禁用，避免冲突
```

## 5. Boot Pool的特殊处理

### 5.1 为什么使用legacy模式？
```bash
# 设置为legacy模式
zfs set mountpoint=legacy bpool/BOOT/debian

# 原因：
1. GRUB兼容性：GRUB可以读取ZFS但有限制
2. 时序控制：确保在根文件系统挂载后再挂载/boot
3. 依赖管理：systemd可以精确控制挂载顺序
```

### 5.2 fstab条目说明
```bash
# Boot pool的fstab条目
bpool/BOOT/debian /boot zfs \
  nodev,relatime,x-systemd.requires=zfs-mount.service,x-systemd.device-timeout=10 0 0

# 参数含义：
- nodev: 不允许设备文件
- relatime: 相对时间更新
- x-systemd.requires=zfs-mount.service: systemd依赖
- x-systemd.device-timeout=10: 设备超时时间
```

## 6. EFI系统分区处理

### 6.1 EFI分区特点
```bash
# EFI分区不是ZFS，而是FAT32文件系统
mkfs.fat -F32 /dev/disk/by-partuuid/xxx

# 挂载层次：
/boot/efi ← EFI分区(FAT32) ← /boot ← Boot Pool(ZFS) ← / ← Root Pool(ZFS)
```

### 6.2 EFI分区的fstab条目
```bash
# 应该添加到fstab（脚本中可能遗漏）
/dev/disk/by-partuuid/xxx /boot/efi vfat defaults 0 1

# 挂载依赖关系：
boot-efi.mount 依赖 boot.mount 依赖 -.mount(根)
```

## 7. 实用命令和检查方法

### 7.1 查看ZFS挂载状态
```bash
# 查看数据集挂载信息
zfs list -o name,mounted,mountpoint,canmount

# 查看池状态
zpool status

# 查看挂载属性
zfs get mountpoint,canmount dataset
```

### 7.2 手动挂载操作
```bash
# 导入池
zpool import poolname

# 挂载特定数据集
zfs mount dataset

# 挂载所有可挂载的数据集
zfs mount -a

# 卸载
zfs umount dataset
```

### 7.3 调试启动问题
```bash
# 检查initramfs中的ZFS支持
lsinitramfs /boot/initrd.img-* | grep zfs

# 检查systemd挂载单元
systemctl list-units | grep mount

# 查看挂载失败日志
journalctl -u boot.mount
```

## 8. 常见配置模式总结

### 8.1 系统池配置（推荐）
```bash
# 根池
zpool create -m none -O mountpoint=none rpool disk
zfs create -o canmount=noauto -o mountpoint=/ rpool/ROOT/debian

# 引导池  
zpool create -m none -O mountpoint=none bpool disk
zfs create -o canmount=noauto -o mountpoint=/boot bpool/BOOT/debian
zfs set mountpoint=legacy bpool/BOOT/debian  # 转为legacy模式
```

### 8.2 数据池配置
```bash
# 数据池（自动挂载）
zpool create -m /data datapool disk
zfs create datapool/documents  # 自动挂载到/data/documents
```

### 8.3 容器池配置
```bash
# 容器根数据集（不挂载）
zfs create -o canmount=off containerPool/containers
zfs create containerPool/containers/container1  # 继承父级设置
```

## 9. 故障排除检查清单

### 9.1 启动失败检查
- [ ] initramfs是否包含ZFS模块
- [ ] 池是否可以正常导入
- [ ] 根数据集的canmount和mountpoint设置
- [ ] /etc/fstab中的ZFS条目是否正确

### 9.2 挂载失败检查  
- [ ] systemd挂载单元状态
- [ ] ZFS服务是否启动
- [ ] 设备路径是否存在（by-partuuid）
- [ ] 文件系统类型是否正确

### 9.3 性能问题检查
- [ ] ZFS ARC内存设置
- [ ] 压缩算法选择
- [ ] ashift设置是否匹配存储设备
