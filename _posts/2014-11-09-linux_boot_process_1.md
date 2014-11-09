# 玩轉 linux 開機流程 - 1
## 寫在前面 
這裡不討論 bootloader, 這塊會跟硬件環境有關.

也不挖掘 linux kernel source.

會討論的只有, bootloader 載入 kernel image and/or initramfs 後發生的事情, 供初學
embedded 開發的人員問題處理的脈絡.

為了方便實驗, 我們使用 kvm 加速實驗過程, 減少實體機器帶來的複雜度.

## linux 開機必須品
要讓 linux 系統正常啟動, 必須準備以下 3 件東西 (除 bootloader 外)

* kernel image
* initial ram disk (initramfs)
* root file system (root fs)

Kernel image 通常放置著 linux kernel 中最重要而無法分離的如 scheduler, syscall,
memory management... 等部份.

Initramfs 是什麼呢? 開機過程需要跟硬件打交道才能完成, 但一般的 distro 無法限制
user 機器上的硬件組合, 又不可能將所有的 drivers 都 build-in 到 kernel
image 中, 這樣產生出來的 kernel image 可能會有上百 M 的大小! Hacker
們就想到了個折衷的方式, 將系統啟動時會用到的 drivers/modules 放到一個 image 中,
以 bootloader 載入到內存中, 並通知 kernel 去裡面找尋需要的 driver, 並在順利開機後,
釋放掉它佔用的內存空間.

Root fs 中放置的是 userland 所需的程序及庫, 所下的命令, 所執行的應用,
都放在裡面.

## 實驗 1
這個實驗利用 kvm 執行現有系統上的 kernel image.

首先, 裝上 kvm
```bash
$ sudo apt-get install -y kvm
```

接著利用系統上的 kernel image 開機, 例如
```bash
$ sudo kvm -kernel /boot/vmlinuz-3.13.0-40-generic
```

如果 kvm 運作正常, linux kernel 應該會 panic, 顯示如下訊息
```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

因為我們並未提供 root fs 給 kvm, 所以當 (panic) 在這個地方是正常的.

## 實驗 2
這個實驗我們來做個最小的 root fs. 最小的 root fs 能有多小? 一個 helloworld
的大小

```C
/* helloworld.c */
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main()
{
    printf("Hello World, I am process %d\n", getpid());

    while(1);

    return 0;
}
```

產生執行檔.

```bash
$ gcc -o helloworld -static helloworld.c
```

因為這個 rootfs 中不會放置其他的 libraries, 所以 helloworld 在 build
的過程中必須要下 -static, 去掉對其他 shared libraries 的相依. 確認一下執行檔無外
部相依

```bash
$ file helloworld 
helloworld: ELF 64-bit LSB  executable, x86-64, version 1 (GNU/Linux), statically linked,
for GNU/Linux 2.6.24, BuildID[sha1]=77ad829f9efc770969e757b467e4c0b066e37db9, not stripped
$ ldd helloworld 
        not a dynamic executable
```

接下來產生 root fs. 如果是一般放在 disk 上的 root fs 可能會有 kernel 中無驅動,
mount 不上造成 boot 失敗的問題. 在這個實驗, 我們先把這個 root fs 放在 initramfs
中. 前面提到過, initramfs 是由 bootloader (或是 kvm) 載入放在 RAM 中的, 因此 kernel
要從中取用東西就會簡單得多, 少掉了如 SATA bus driver, SATA disk driver, file
system 等需求.

```bash
$ echo init | cpio -H newc -o | gzip -c >initramfs
$ file initramfs 
initramfs: gzip compressed data, from Unix, last modified: Sun Nov  9 14:51:59
2014
```

執行 kvm
```bash
$ sudo kvm -kernel /boot/vmlinuz-3.13.0-40-generic -initrd initramfs
```

如果順利的話, kvm 會 boot 到印出 "Hello World! I am process 1" 後停止, 因為
hellworld 是 userspace 的第一個 process, 因此 process ID 一定為 1.
