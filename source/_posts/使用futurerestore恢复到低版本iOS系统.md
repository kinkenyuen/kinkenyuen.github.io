---
title: 使用futurerestore恢复到低版本iOS系统
date: 2020-01-16 19:45:12
tags: iOS
categories: iOS
---

# 前言
一般而言，iOS设备上的固件恢复需要配合Apple服务器进行校验，Apple停止公开验证某个固件版本时，iOS设备就不能从高版本恢复到停止验证的版本。

# 前置条件
1. 具备解锁nvram，写入generator的可能
2. 根据设备的唯一码备份了对应的`SHSH2`文件
3. 当前最新固件SEP兼容需要降级的目标版本固件

## generator&nonce
generator是记录在shsh2文件中的一串值，这串值对应着一个nonce。nonce是一个只能使用一次的随机数。它在认证协议中用于阻止重放攻击。

## 原理
>iOS/iTunes 在更新设备固件的过程中，会将设备的 ECID，系统版本等信息，以及一个一次使用的 Nonce 发送给 Apple 的验证服务器，服务器在校验通过后，会返回校验结果给 iOS/iTunes，结果使用非对称算法加密，在没有私钥的情况下无法解密，也无法伪造。

>但是，我们可以将校验结果保存下来，之后 Apple 不再提供此版本校验的时候（假设不考虑 SEP 兼容性），在越狱后通过 nvram 固定 nonce 为此校验结果使用的，来重放校验过程，实现 iOS 系统降级/更新到不提供验证的版本。

简单理解，备份的shsh2文件对应着一个nonce，在恢复到低版本的时候，先固定shsh2对应的nonce，然后绕过校验进行固件恢复

# 实例

## 环境&工具
* 使用CheckRa1n越狱后的iPhone 6 iOS 12.4.4
* MacOS 10.15
* nonce固定工具Generator Auto Setter
* [futurerestore](https://github.com/tihmstar/futurerestore)(latest version is recommend)

## 准备工作
简单而言分为两步，第一固定nonce，第二连接设备使用futurerestore刷写

### 固定nonce

设备越狱后添加repo:[halo-michale.github.io/repo/](halo-michale.github.io/repo/),安装插件Generator Auto Setter，默认会写入G值0x1111111111111111,我需要降到12.4，找到自己的shsh2文件里的G值为0xadcedaa4dc76c6f8,于是ssh root到越狱手机

```sh
Ken:~ root# setgenerator 0xadcedaa4dc76c6f8
```
```sh
arm_pgshift: 12
tfp0: 0xa03
kbase: 0xfffffff00f404000
kslide: 0x8400000
sec_cstring_start: 0xfffffff00f607aa0, sec_cstring_sz: 0x24f9f9
sec_text_start: 0xfffffff00fa68000, sec_text_sz: 0x12ab540
allproc: 0xfffffff010e546e8
our_task: 0xfffffff08bc7d920
nonce_serv: 0x1307
nonce_conn: 0xb07
ipc_port: 0xfffffff08d7f32c8
nonce_object: 0xfffffff08a3b89a0
boot_nonce_os_symbol: 0xfffffff08a330340
nvram_serv: 0xb0f
ipc_port: 0xfffffff08b667df0
nvram_object: 0xfffffff08a291d00
of_dict: 0xfffffff08a3bca50
os_dict_cnt: 0xc
os_dict_entry_ptr: 0xfffffff08b593e60
key: 0xfffffff08a342aa0, value: 0xfffffff08e697c90
key: 0xfffffff08a3423c0, value: 0xfffffff08bff84e0
key: 0xfffffff08a345e60, value: 0xfffffff08a3bc7e0
key: 0xfffffff08a333f00, value: 0xfffffff08a3bc960
key: 0xfffffff08a3459c0, value: 0xfffffff08a3bc870
key: 0xfffffff08a353120, value: 0xfffffff08a3bcab0
key: 0xfffffff08a353060, value: 0xfffffff08a3bc690
key: 0xfffffff08a353fa0, value: 0xfffffff08a3bc6c0
key: 0xfffffff08a353f00, value: 0xfffffff08a3ddf90
key: 0xfffffff08a353f40, value: 0xfffffff08a3bca80
key: 0xfffffff08a353200, value: 0xfffffff08a3531e0
key: 0xfffffff08a330340, value: 0xfffffff08be631c0
os_string: 0xfffffff08be631c0
string_ptr: 0xfffffff08b6a5e40
Set nonce to 0xadcedaa4dc76c6f8
```

### 恢复固件

```sh
./futurerestore -t 2375331941521446_iPhone7,2_12.4-16G77_31decc9d1a18ca4192f886be692e2b6d5b6118d7.shsh2 --latest-sep --latest-baseband iPhone_4.7_12.4_16G77_Restore.ipsw
```

```plain
➜  Downgrade ./futurerestore -t 2375331941521446_iPhone7,2_12.4-16G77_31decc9d1a18ca4192f886be692e2b6d5b6118d7.shsh2 --latest-sep --latest-baseband iPhone_4.7_12.4_16G77_Restore.ipsw
Version: 81b98e0425e17250cc83d5badaf9a8cc6399f481 - 245
Libipatcher version: 3159a387584e352f690cca859e013c3a4683f3e8 - 69
Odysseus support: yes
[INFO] 64-bit device detected
futurerestore init done
reading signing ticket 2375331941521446_iPhone7,2_12.4-16G77_31decc9d1a18ca4192f886be692e2b6d5b6118d7.shsh2 is done
Found device iPhone7,2 n61ap
user specified to use latest signed SEP (WARNING, THIS CAN CAUSE A NON-WORKING RESTORE)
[TSSC] opening firmware.json
[DOWN] downloading file https://api.ipsw.me/v2.1/firmwares.json/condensed
[TSSC] selecting latest iOS: 12.4.4
[TSSC] got firmware URL for iOS 12.4.4 build 16G140
[TSSC] opening Buildmanifest for iPhone7,2_12.4.4
100 [===================================================================================================>]
downloading SEP

100 [===================================================================================================>]
[TSSC] opening /tmp/futurerestore/sepManifest.plist
[TSSR] User specified not to request a baseband ticket.
Request URL set to https://gs.apple.com/TSS/controller?action=2
Sending TSS request attempt 1... response successfully received
user specified to use latest signed baseband (WARNING, THIS CAN CAUSE A NON-WORKING RESTORE)
downloading baseband

100 [===================================================================================================>]
[TSSC] opening /tmp/futurerestore/basebandManifest.plist
[TSSR] User specified to request only a baseband ticket.
Request URL set to https://gs.apple.com/TSS/controller?action=2
Sending TSS request attempt 1... response successfully received
Found device in Normal mode
Entering recovery mode...
INFO: device serial number is F78PGK5UG5MT
Found device in Recovery mode
Identified device as n61ap, iPhone7,2
Extracting BuildManifest from iPSW
Product version: 12.4
Product build: 16G77 Major: 16
Device supports IMG4: true
Got ApNonce from device: 31 de cc 9d 1a 18 ca 41 92 f8 86 be 69 2e 2b 6d 5b 61 18 d7
checking APTicket to be valid for this restore...
Verified ECID in APTicket matches device ECID
checking APTicket to be valid for this restore...
Verified ECID in APTicket matches device ECID
Verified APTicket to be valid for this restore
Variant: Customer Erase Install (IPSW)
This restore will erase your device data.
Extracting filesystem from iPSW
[==================================================] 100.0%
Extracting iBEC.n61.RELEASE.im4p...
Personalizing IMG4 component iBEC...
Sending iBEC (731534 bytes)...
waiting for device to reconnect...
Getting SepNonce in recovery mode... ea 37 e5 69 22 48 c6 ac 9f f8 1d 3e 78 67 97 87 e9 f9 ce aa
Getting ApNonce in recovery mode... 31 de cc 9d 1a 18 ca 41 92 f8 86 be 69 2e 2b 6d 5b 61 18 d7
[WARNING] Setting bgcolor to green! If you don't see a green screen, then your device didn't boot iBEC correctly
Recovery Mode Environment:
iBoot build-version=iBoot-4513.270.14
iBoot build-style=RELEASE
Sending RestoreLogo...
Extracting applelogo@2x~iphone.im4p...
Personalizing IMG4 component RestoreLogo...
Sending RestoreLogo (12334 bytes)...
Extracting 048-78047-092.dmg.trustcache...
Personalizing IMG4 component RestoreTrustCache...
Sending RestoreTrustCache (9681 bytes)...
ramdisk-size=0x10000000
Extracting 048-78047-092.dmg...
Personalizing IMG4 component RestoreRamDisk...
Sending RestoreRamDisk (91608779 bytes)...
Extracting DeviceTree.n61ap.im4p...
Personalizing IMG4 component RestoreDeviceTree...
Sending RestoreDeviceTree (125713 bytes)...
Extracting kernelcache.release.iphone7...
Personalizing IMG4 component RestoreKernelCache...
Sending RestoreKernelCache (14069235 bytes)...
Trying to fetch new signing tickets
Request URL set to https://gs.apple.com/TSS/controller?action=2
Sending TSS request attempt 1... response successfully received
Received signing tickets
About to restore device...
Waiting for device...
Device fa7290e4f299aff884b5eb6febce2643d11d092d is now connected in restore mode...
Connecting now...
Connected to com.apple.mobile.restored, version 15
Device fa7290e4f299aff884b5eb6febce2643d11d092d has successfully entered restore mode
Hardware Information:
BoardID: 6
ChipID: 28672
UniqueChipID: 2375331941521446
ProductionMode: true
Starting FDR listener thread
About to send RootTicket...
Sending RootTicket now...
Done sending RootTicket
Waiting for NAND (28)
About to send NORData...
Found firmware path Firmware/all_flash
Getting firmware manifest from build identity
Extracting LLB.n61.RELEASE.im4p...
Personalizing IMG4 component LLB...
Extracting applelogo@2x~iphone.im4p...
Personalizing IMG4 component AppleLogo...
Extracting batterycharging0@2x~iphone.im4p...
Personalizing IMG4 component BatteryCharging0...
Extracting batterycharging1@2x~iphone.im4p...
Personalizing IMG4 component BatteryCharging1...
Extracting batteryfull@2x~iphone.im4p...
Personalizing IMG4 component BatteryFull...
Extracting batterylow0@2x~iphone.im4p...
Personalizing IMG4 component BatteryLow0...
Extracting batterylow1@2x~iphone.im4p...
Personalizing IMG4 component BatteryLow1...
Extracting glyphplugin@1334~iphone-lightning.im4p...
Personalizing IMG4 component BatteryPlugin...
Extracting DeviceTree.n61ap.im4p...
Personalizing IMG4 component DeviceTree...
Extracting recoverymode@1334~iphone-lightning.im4p...
Personalizing IMG4 component RecoveryMode...
Extracting iBoot.n61.RELEASE.im4p...
Personalizing IMG4 component iBoot...
Personalizing IMG4 component RestoreSEP...
Personalizing IMG4 component SEP...
Sending NORData now...
Done sending NORData
Checking filesystems (15)
Checking filesystems (15)
About to send FDR Trust data...
Sending FDR Trust data now...
Done sending FDR Trust Data
Unmounting filesystems (29)
Unmounting filesystems (29)
Unmounting filesystems (29)
Unmounting filesystems (29)
Creating partition map (11)
Creating filesystem (12)
About to send filesystem...
Connected to ASR
Validating the filesystem
Filesystem validated
Sending filesystem now...
[==================================================] 100.0%
Done sending filesystem
Verifying restore (14)
[==================================================] 100.0%
Checking filesystems (15)
Checking filesystems (15)
Checking filesystems (15)
Checking filesystems (15)
Mounting filesystems (16)
Mounting filesystems (16)
Mounting filesystems (16)
About to send KernelCache...
Extracting kernelcache.release.iphone7...
Personalizing IMG4 component KernelCache...
Sending KernelCache now...
Done sending KernelCache
Installing kernelcache (27)
About to send DeviceTree...
Extracting DeviceTree.n61ap.im4p...
Personalizing IMG4 component DeviceTree...
Sending DeviceTree now...
Done sending DeviceTree
Certifying Savage (61)
Flashing firmware (18)
[==================================================] 100.0%
Unknown operation (36)
About to send FUD data...
Found FUD component 'RestoreTrustCache'
Extracting 048-78047-092.dmg.trustcache...
Personalizing IMG4 component RestoreTrustCache...
Found FUD component 'StaticTrustCache'
Extracting 048-76490-092.dmg.trustcache...
Personalizing IMG4 component StaticTrustCache...
Sending FUD data now...
Done sending FUD data
Updating gas gauge software (47)
Updating gas gauge software (47)
Updating Stockholm (55)
Unknown operation (36)
About to send FUD data...
Found FUD component 'RestoreTrustCache'
Extracting 048-78047-092.dmg.trustcache...
Personalizing IMG4 component RestoreTrustCache...
Found FUD component 'StaticTrustCache'
Extracting 048-76490-092.dmg.trustcache...
Personalizing IMG4 component StaticTrustCache...
Sending FUD data now...
Done sending FUD data
Updating baseband (19)
About to send BasebandData...
Sending Baseband TSS request...
Request URL set to https://gs.apple.com/TSS/controller?action=2
Sending TSS request attempt 1... response successfully received
Received Baseband SHSH blobs
Sending BasebandData now...
Done sending BasebandData
Updating Baseband in progress...
About to send BasebandData...
Sending BasebandData now...
Done sending BasebandData
Updating Baseband completed.
Updating SE Firmware (59)
Fixing up /var (17)
Creating system key bag (50)
Modifying persistent boot-args (25)
Unmounting filesystems (29)
Unmounting filesystems (29)
Unmounting filesystems (29)
Unmounting filesystems (29)
Got status message
Status: Restore Finished
Cleaning up...
DONE
Done: restoring succeeded.
```

