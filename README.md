我的guest版本是Ubuntu 22.04.5 LTS x86_64，内核 5.15.0-170-generic

wsl是Ubuntu 24.04.2 LTS on Windows 10 x86_64，内核 6.6.87.2-microsoft-standard-WSL2

## 准备编译环境

Guest VM内执行：

```bash
sudo apt update

## 安装构建依赖
sudo apt install -y \
    bison \
    flex \
    libssl-dev \
    libelf-dev \
    linux-headers-$(uname -r) \
    pahole \
    dwarves \
    build-essential \
    kmod \
    cpio \
    rsync \
    bc \
    kmod

# 看下头文件全了没
ls -l /usr/src/linux-headers-$(uname -r)/
ls -l /lib/modules/$(uname -r)/build  # 应该指向头文件目录

```

## 获取指定版本内核源码

需要注意version magic对不上会拒绝加载编译出来的内核模块。

```bash
sudo apt install linux-source-5.15.0 -y

cd /usr/src
sudo tar -xf linux-source-5.15.0.tar.bz2 # 解压

# 然后把这个仓库patch进去
# 略

```

## 编译内核模块

```bash
cd /usr/src/linux-source-5.15.0/drivers/nvme/host
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

```

## 加载模块

```bash
cd /usr/src/linux-source-5.15.0/drivers/nvme/host
sudo rmmod nvme
sudo insmod ./nvme.ko
```

## 测试

fio随机写

```bash
sudo fio --name=test --filename=/dev/nvme0n1 \
    --direct=1 --rw=randwrite --bs=4k \
    --ioengine=libaio --iodepth=32 \
    --numjobs=4 --size=1G \
    --group_reporting
```

fio全盘随机读

```bash
sudo fio --name=test --filename=/dev/nvme0n1 \
    --direct=1 --rw=randread --bs=4k \
    --ioengine=libaio --iodepth=32 \
    --numjobs=4 --time_based --runtime=30 \
    --group_reporting
```

挂载ext4

```bash
sudo lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT /dev/nvme0n1

# 分区表
sudo parted -s /dev/nvme0n1 mklabel gpt
sudo parted -s /dev/nvme0n1 mkpart primary ext4 1MiB 100%
sudo partprobe /dev/nvme0n1

sudo mkfs.ext4 -F /dev/nvme0n1p1 # 格式化

# 挂载
sudo mkdir -p /mnt/femu
sudo mount /dev/nvme0n1p1 /mnt/femu

# 随便写点文件测试
echo "hello world" | sudo tee /mnt/femu/hello.txt > /dev/null
sudo sync
sudo cat /mnt/femu/hello.txt
```

## 常用命令

windows ps终端远程连接到guest

```bash
ssh -p 8080 g122622@localhost
```

看内核日志

```bash
sudo dmesg | tail -20

# 实时
sudo dmesg -w
```
