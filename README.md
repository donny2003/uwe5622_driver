# uwe5622_driver

uwe5622的内核驱动

从[linux-orangepi](https://github.com/orangepi-xunlong/linux-orangepi/tree/orange-pi-6.6-rk35xx)中复制了6.6版本，升级到了6.11

~~买开发板之前一定要确认关键驱动有没有进内核主线~~

by 2025-11-07 update 修改uwe5622源代码以适配6.1内核，以下为手动修改日志
#编译命令
make ARCH=arm64 -C /lib/modules/$(uname -r)/build M=$PWD CONFIG_RK_WIFI_DEVICE_UWE5622=y CONFIG_WLAN_UWE5622=y CONFIG_TTY_OVERY_SDIO=y modules
# 备份文件
cp /root/uwe5622_driver/unisocwcn/platform/wcn_boot.c /root/uwe5622_driver/unisocwcn/platform/wcn_boot.c.backup
# 编辑文件
nano /root/uwe5622_driver/unisocwcn/platform/wcn_boot.c
// 修改前：
static void marlin_remove(struct platform_device *pdev)
// 修改后：
static int marlin_remove(struct platform_device *pdev)
// 在函数最后添加：
return 0;

# 备份文件
cp /root/uwe5622_driver/unisocwcn/platform/wcn_log.c /root/uwe5622_driver/unisocwcn/platform/wcn_log.c.backup
nano /root/uwe5622_driver/unisocwcn/platform/wcn_log.c
#将代码中的：class_create("slog_wcn") 修改为：class_create(THIS_MODULE, "slog_wcn")

# 备份文件
cp /root/uwe5622_driver/unisocwifi/cmdevt.c /root/uwe5622_driver/unisocwifi/cmdevt.c.backup

# 编辑文件
nano /root/uwe5622_driver/unisocwifi/cmdevt.c
#找到第3307行附近的 cfg80211_ch_switch_notify 调用，将其从修改为：
因此，我们可以这样修改条件编译：
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(6,1,0))
cfg80211_ch_switch_notify(vif->ndev, &chandef, 0);
#elif (LINUX_VERSION_CODE >= KERNEL_VERSION(5,19,2))
cfg80211_ch_switch_notify(vif->ndev, &chandef, 0, 0);
#else
cfg80211_ch_switch_notify(vif->ndev, &chandef);
#endif

# 备份文件
cp /root/uwe5622_driver/unisocwifi/npi.c /root/uwe5622_driver/unisocwifi/npi.c.backup

# 回调函数应该使用 struct genl_ops 而不是 struct genl_split_ops
nano /root/uwe5622_driver/unisocwifi/npi.c
找到struct genl_split_ops修改为： struct genl_ops

# 备份文件
cp /root/uwe5622_driver/unisocwifi/wl_core.c /root/uwe5622_driver/unisocwifi/wl_core.c.backup
修改
nano /root/uwe5622_driver/unisocwifi/wl_core.c
#找到sprdwl_remove函数，修改返回值

// 修改前：
static void sprdwl_remove(struct platform_device *pdev)
// 修改后：
static int sprdwl_remove(struct platform_device *pdev)
// 在函数最后添加：
return 0;

#以下这些代码都需要同样更正
 find /root/uwe5622_driver -name "*.c" -exec grep -l "\.remove.*=" {} \; | xargs grep -l "static void.*remove"

# 备份文件
cp /root/uwe5622_driver/tty-sdio/tty.c /root/uwe5622_driver/tty-sdio/tty.c.backup

nano /root/uwe5622_driver/tty-sdio/tty.c
#修改方法，查找 mtty_remove，找到对应的函数体就可以找到修改函数返回值

#修复这些警告（可选）
# 备份文件
cp /root/uwe5622_driver/unisocwcn/sdio/sdiohal_main.c /root/uwe5622_driver/unisocwcn/sdio/sdiohal_main.c.backup

# 编辑文件
nano /root/uwe5622_driver/unisocwcn/sdio/sdiohal_main.c

# 在这些函数前添加 __maybe_unused 属性：
static int __maybe_unused sdiohal_suspend(struct device *dev)
static int __maybe_unused sdiohal_resume(struct device *dev)

修复方法（可选，编译完成后）
# 备份文件
cp /root/uwe5622_driver/unisocwifi/main.c nano /root/uwe5622_driver/unisocwifi/main.c.backup
# 编辑文件
nano /root/uwe5622_driver/unisocwifi/main.c
找到第947行附近的代码，添加类型转换：
// 修改前：
memcpy(dev->dev_addr, sa->sa_data, ETH_ALEN);
// 修改后：
memcpy((void *)dev->dev_addr, sa->sa_data, ETH_ALEN);

同样修复第952行的相同问题：
// 修改前：
memcpy(dev->dev_addr, vif->mac, ETH_ALEN);
// 修改后：
memcpy((void *)dev->dev_addr, vif->mac, ETH_ALEN);

==========================================================================
# 最后的手段：强制编译，忽略未解析的符号（不推荐，但可以测试）注意：这可能会产生不稳定的驱动
make ARCH=arm64 -C /lib/modules/$(uname -r)/build M=$PWD CONFIG_RK_WIFI_DEVICE_UWE5622=y CONFIG_WLAN_UWE5622=y CONFIG_TTY_OVERY_SDIO=y modules_install
======================================================
#不建议执行，与下面的编辑文件二选一
sed -i '/case DEL_LUT_INDEX:/{n;n;s/case ADD_LUT_INDEX:/\/* fall through *\/\n&/}' /root/uwe5622_driver/unisocwifi/wl_intf.c

# 编辑文件
nano /root/uwe5622_driver/unisocwifi/wl_intf.c
找到第1570行附近的代码，在 sprdwl_dis_flush_txlist(intf, i); 后面添加明确的 fallthrough 注释：
case DEL_LUT_INDEX:
    sprdwl_dis_flush_txlist(intf, i);
    /* fall through */
case ADD_LUT_INDEX:
    // ... 其他代码
    
这个函数通常用于设置进程调度策略。我们可以修改驱动代码来避免使用它：
==========================================================================
# 查找使用 sched_setscheduler 的文件
find /root/uwe5622_driver -name "*.c" -o -name "*.h" | xargs grep -l "sched_setscheduler" 2>/dev/null
/root/uwe5622_driver/unisocwcn/usb/wcn_usb_rx_tx.c
/root/uwe5622_driver/unisocwcn/pcie/edma_engine.c
/root/uwe5622_driver/unisocwcn/sdio/sdiohal_rx.c
/root/uwe5622_driver/unisocwcn/sdio/sdiohal_tx.c

# 备份上述文件
find /root/uwe5622_driver -name "*.c" -exec grep -l "sched_setscheduler" {} \; | xargs -I {} cp {} {}.backup
注释上述源代码中的 sched_setscheduler调用
sed -i 's/^[[:space:]]*sched_setscheduler(/\/\/ sched_setscheduler(/g' /root/uwe5622_driver/unisocwcn/usb/wcn_usb_rx_tx.c
sed -i 's/^[[:space:]]*sched_setscheduler(/\/\/ sched_setscheduler(/g' /root/uwe5622_driver/unisocwcn/pcie/edma_engine.c
sed -i 's/^[[:space:]]*sched_setscheduler(/\/\/ sched_setscheduler(/g' /root/uwe5622_driver/unisocwcn/sdio/sdiohal_rx.c
sed -i 's/^[[:space:]]*sched_setscheduler(/\/\/ sched_setscheduler(/g' /root/uwe5622_driver/unisocwcn/sdio/sdiohal_tx.c
#下句不执行否则会将sched_setscheduler中的参数替换掉
sed -i 's/sched_setscheduler([^)]*);/\/\/ sched_setscheduler 调用已被注释掉/g' /root/uwe5622_driver/unisocwcn/usb/wcn_usb_rx_tx.c

