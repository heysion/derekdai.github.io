# 玩轉 linux 開機流程
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
user 機器上的硬件組合, 又不可能將所有的 driver 都 build-in 到 kernel
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
這個實驗我們來做個最小的 root fs. 最小的 root fs 能多少? 
