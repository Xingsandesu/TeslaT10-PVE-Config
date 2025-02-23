# Tesla T10 PVE 服务器配置指南

## 1. 配置 APT 源
```bash
cat >/etc/apt/sources.list<<EOF
deb http://mirrors.ustc.edu.cn/debian/ bookworm main non-free-firmware
deb-src http://mirrors.ustc.edu.cn/debian/ bookworm main non-free-firmware
deb https://mirrors.ustc.edu.cn/debian-security bookworm-security main contrib
deb http://mirrors.ustc.edu.cn/debian/ bookworm-updates main non-free-firmware
deb-src http://mirrors.ustc.edu.cn/debian/ bookworm-updates main non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/proxmox/debian bookworm pve-no-subscription
EOF

# 备份原有源文件
mv /etc/apt/sources.list.d/ceph.list /etc/apt/sources.list.d/ceph.list.bak
mv /etc/apt/sources.list.d/pve-enterprise.list /etc/apt/sources.list.d/pve-enterprise.list.bak
```

## 2. 安装系统依赖
```bash
apt-get install -y git build-essential dkms mdevctl
apt install pve-headers-$(uname -r)
```

## 3. 网卡配置
将网卡 MAC 地址绑定到固定接口名称:
```bash
# 替换 xxx 为实际的 MAC 地址
echo 'SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="xxx", NAME="eth0"' | tee /etc/udev/rules.d/71-net-name-by-mac.rules
echo 'SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="xxx", NAME="eth1"' | tee /etc/udev/rules.d/72-net-name-by-mac.rules

# 编辑网络配置文件
vim /etc/network/interfaces
```

## 4. 存储配置
将 LVM 转换为目录存储:
```bash
lvremove pve/data
lvextend -l +100%FREE -r pve/root
pvesm remove local-lvm
mkdir /local-vm
pvesm add dir local-vm --content images,iso,vztmpl,backup,rootdir,snippets -preallocation off --path /local-vm
```

## 5. 内核管理（可选）
```bash
# 安装指定版本内核
apt-get update && apt-get install proxmox-kernel-6.8.12-4-pve
proxmox-boot-tool kernel list
proxmox-boot-tool kernel pin 6.8.12-4-pve
proxmox-boot-tool refresh
```

## 6. NVIDIA 驱动安装
```bash
bash NVIDIA-Linux-x86_64-550.144.02-vgpu-kvm-custom.run -m kernel
```

## 7. IOMMU 配置
```bash
# 启用 IOMMU
if [ `grep -c "intel_iommu=on" /etc/default/grub` -ne '1' ];then
  sed -i 's/quiet/quiet intel_iommu=on/' /etc/default/grub
  update-grub
fi

# 启用 vfio 并禁用 nouveau
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u -k all
rmmod nouveau
```

## 8. vGPU 解锁配置
```bash
# 克隆必要的仓库
git clone https://gitlab.com/polloloco/vgpu-proxmox.git
cd /opt
git clone https://github.com/mbilker/vgpu_unlock-rs.git

# 安装 Rust 环境
curl https://sh.rustup.rs -sSf | sh -s -- -y --profile minimal
source $HOME/.cargo/env

# 编译 vGPU 解锁工具
cd vgpu_unlock-rs/
cargo build --release

# 配置系统服务
mkdir /etc/vgpu_unlock
touch /etc/vgpu_unlock/profile_override.toml
mkdir /etc/systemd/system/{nvidia-vgpud.service.d,nvidia-vgpu-mgr.service.d}
echo -e "[Service]\nEnvironment=LD_PRELOAD=/opt/vgpu_unlock-rs/target/release/libvgpu_unlock_rs.so" > /etc/systemd/system/nvidia-vgpud.service.d/vgpu_unlock.conf
echo -e "[Service]\nEnvironment=LD_PRELOAD=/opt/vgpu_unlock-rs/target/release/libvgpu_unlock_rs.so" > /etc/systemd/system/nvidia-vgpu-mgr.service.d/vgpu_unlock.conf
```

## 9. vGPU 配置文件
```bash
cat >/etc/vgpu_unlock/profile_override.toml<<EOF
[profile.nvidia-262]
framebuffer = 0x3B0000000
pci_id = 0x1EB81310
pci_device_id = 0x1EB8
EOF
```

## 其他工具

### 风扇控制
- T10 风扇控制工具: [https://github.com/Xingsandesu/t10-fan-control](https://github.com/Xingsandesu/t10-fan-control)

## 许可证
MIT License

### 作者致谢
- Nanyo
- Fushinn
- [https://github.com/Xingsandesu/](https://github.com/Xingsandesu/)
