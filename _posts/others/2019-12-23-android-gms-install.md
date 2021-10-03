---
title: "android 安装google play"
tags: [android, gms, "google play"]
---

由于google退出中国的原因，国内的 `andriod` 手机很多都移除了对 `google play` 的支持。

很多的应用都需要有 `google play` 才能运行，在没有 `google play` 的情况下，就算通过 `apk` 的方式安装了应用。往往也会出现无法登陆，或者白屏的情况。

研究了一下要成功的使用 `google play`，大概需要安装一下几个部分：

- gms [download](https://apkpure.com/gms-installer/com.huawei.gmsinstaller)
- google play store [download](https://apkpure.com/google-play-store/com.android.vending)
- google play services [download](https://apkpure.com/search?q=google+play+services)
- Google Services Framework [download](https://apkpure.com/google-services-framework/com.google.android.gsf)

其中 `gms` 是基础，我安装的时候实现安装了 `google play store`, `google play services` 和 `Google Services Framework`。
最后在安装的 `GMS`, 但是好像安装 `GMS` 的时候默认就会安装其中的几个。这个没有继续试验。

另外， 安装 `GMS` 的时候会把一些安装了的google的应用删除后重新安装，比如 `google play store` ， `google keep` 等。这个重装应该是在安装 `GMS` 的时候手机重启后进行重装，所以，如果要保证能够重装成功的话，需要保证在重启后手机依然能够连接上google的服务器。不然重装的程序不能能成功安装（重启时 `GMS` 已经成功安装）。对于没有重装成功的应用，自己在手动重装一次即可。

# 关键点

## 启用未知来源安装
有些手机会禁用未知来源的软件安装，所以在使用 `apk` 进行安装的时候可能出现没有安装选项的问题，需要先进行一下权限设置：

1. “设置” --> "高级设置"
2. "安全" --> "启用未知来源安装"

## 联网

# References

- [Install Core GMS Packages on Huawei Chinese Phone to Run Google Apps](https://itechify.com/2018/10/16/install-core-gsm-packages-huawei-chinese-phones/)
- [How to Download and Install Google Play Store on Huawei Chinese phones](https://huaweiadvices.com/download-install-google-play-store-on-huawei/)
