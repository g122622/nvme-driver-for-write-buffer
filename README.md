# 1. 性能测试

# 2. 构建&运行

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

## 阻止内核自动更新

内核更新之后可能破坏驱动兼容性，要锁一下内核版本

```bash
sudo apt-mark hold linux-image-5.15.0-171-generic
sudo apt-mark hold linux-headers-5.15.0-171-generic
sudo apt-mark hold linux-image-generic
sudo apt-mark hold linux-headers-generic

# 验证下
apt-mark showhold
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

自动化脚本一键测试

```bash
cd /usr/src/linux-source-5.15.0/drivers/nvme
sudo ./run_test.sh
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

# 权限多给点
sudo chown -R $USER:$USER /mnt/femu

# 随便写点文件测试
echo "hello world" | sudo tee /mnt/femu/hello.txt > /dev/null
sudo sync
sudo cat /mnt/femu/hello.txt
```

测试脚本（js）

```js
#!/usr/bin/env node

/**
 * Node.js v12 兼容性文件 IO 完整性测试脚本
 * 环境要求：Node.js v12+
 */

const fs = require('fs').promises;
const fsSync = require('fs');
const path = require('path');
const crypto = require('crypto');
const assert = require('assert');

// 配置项
const CONFIG = {
    smallSize: 1024,             // 1KB
    largeSize: 50 * 1024 * 1024, // 50MB
    baseDir: __dirname
};

// 工具：生成随机 Buffer
function generateRandomData(size) {
    return crypto.randomBytes(size);
}

// 工具：格式化字节大小
function formatBytes(bytes) {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
}

// 工具：清理文件（如果存在）
async function safeUnlink(filePath) {
    try {
        await fs.access(filePath);
        await fs.unlink(filePath);
        console.log(`  [Cleanup] 已清理文件：${path.basename(filePath)}`);
    } catch (e) {
        // 文件不存在则忽略
    }
}

// 核心测试逻辑
async function runTestCase(label, size) {
    const timestamp = Date.now();
    const fileName = `test_${label}_${timestamp}.dat`;
    const filePath = path.join(CONFIG.baseDir, fileName);
    
    console.log(`\n----------------------------------------`);
    console.log(`开始测试：[${label}] (大小：${formatBytes(size)})`);
    console.log(`文件路径：${filePath}`);
    console.log(`----------------------------------------`);

    let originalData = null;
    let appendData = null;
    let expectedFinalData = null;

    try {
        // 1. 准备数据
        console.log(`[1/6] 生成随机数据...`);
        originalData = generateRandomData(size);
        appendData = generateRandomData(size); // 追加同样大小的数据进行测试
        expectedFinalData = Buffer.concat([originalData, appendData]);

        // 2. 写入文件
        console.log(`[2/6] 写入文件...`);
        const startWrite = Date.now();
        await fs.writeFile(filePath, originalData);
        console.log(`  -> 写入完成，耗时：${Date.now() - startWrite}ms`);

        // 3. 读取并验证完整性 (Write Check)
        console.log(`[3/6] 读取并验证写入完整性...`);
        const readData1 = await fs.readFile(filePath);
        assert.strictEqual(readData1.length, originalData.length, '第一次读取长度不匹配');
        assert.strictEqual(Buffer.compare(readData1, originalData), 0, '第一次读取内容不一致 (Buffer 比较失败)');
        console.log(`  -> 验证通过 (Hash 一致)`);

        // 4. 追加数据
        console.log(`[4/6] 追加数据...`);
        const startAppend = Date.now();
        await fs.appendFile(filePath, appendData);
        console.log(`  -> 追加完成，耗时：${Date.now() - startAppend}ms`);

        // 5. 读取并验证完整性 (Append Check)
        console.log(`[5/6] 读取并验证追加完整性...`);
        const readData2 = await fs.readFile(filePath);
        assert.strictEqual(readData2.length, expectedFinalData.length, '第二次读取长度不匹配');
        assert.strictEqual(Buffer.compare(readData2, expectedFinalData), 0, '第二次读取内容不一致 (Buffer 比较失败)');
        console.log(`  -> 验证通过 (Hash 一致)`);

        // 6. 删除文件
        console.log(`[6/6] 删除文件...`);
        await fs.unlink(filePath);

        // 7. 验证删除
        try {
            await fs.access(filePath);
            throw new Error('文件删除失败：文件仍然存在');
        } catch (e) {
            if (e.code === 'ENOENT') {
                console.log(`  -> 删除验证通过 (文件确认不存在)`);
            } else {
                throw e;
            }
        }

        console.log(`✅ [${label}] 测试全部通过！`);
        return true;

    } catch (error) {
        console.error(`\n❌ [${label}] 测试失败！`);
        console.error(`错误信息：${error.message}`);
        console.error(`堆栈：${error.stack}`);
        
        // 失败时尝试清理残留文件
        await safeUnlink(filePath);
        return false;
    }
}

// 主入口
async function main() {
    console.log(`Node.js 版本：${process.version}`);
    console.log(`当前工作目录：${process.cwd()}`);
    console.log(`测试目录：${CONFIG.baseDir}`);

    let allPassed = true;

    // 执行小文件测试
    const smallResult = await runTestCase('SMALL_1KB', CONFIG.smallSize);
    if (!smallResult) allPassed = false;

    // 执行大文件测试
    const largeResult = await runTestCase('LARGE_50MB', CONFIG.largeSize);
    if (!largeResult) allPassed = false;

    console.log(`\n========================================`);
    if (allPassed) {
        console.log(`🎉 所有测试项目通过！`);
        process.exitCode = 0;
    } else {
        console.log(`💥 部分测试项目失败！`);
        process.exitCode = 1;
    }
    console.log(`========================================\n`);
}

// 执行主函数 (Node 12 不支持顶层 await，需包裹)
main().catch(err => {
    console.error('脚本执行发生未捕获错误:', err);
    process.exitCode = 1;
});

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
