---
title: 对木马LummaC Stealer样本极简的一次分析
date: 2023-11-22 17:47:43
tags:
---


本文带你通过公开的网站分析捕获的恶意程序。

## 起因

电脑被人借用时，下盗版软件下到一个一看就是下崽器的东西，但是估计他没多想，我也没多想。2023-10-15

## 铺垫

火绒日志被清空

谷歌报恶意软件活动 强制退出

GitHub被标记为spam

数字签名的一些驱动被篡改

## 高潮

Steam被盗 2023-11-07

邮箱被改绑

追回：找客服撕逼

事实上，在邮箱里能找到“已删除”的邮件：

![一分钟内完成验证改绑与删除邮件](1.png)

一分钟内完成验证改绑与删除邮件

![Untitled](2.png)

![Untitled](3.png)

## 诱因&基本情况

病毒文件来源：

[https://[REDACTED].click/cgi-sys/defaultwebpage.cgi](https://[REDACTED].com/guitar-pro-crack/)

→https://[REDACTED].click/?i=Guitar-Pro-8-3-3-Crack-With-License-Key-2023-Free-Download&u=1699615197&t=17

→https://[REDACTED].cfd/

→Passwd.2024.Setap.rar

> 💡 2023-11-11 Update: 文件被更新
→<https://href.li/?https://[REDACTED].cfd/?h=rfbnUPMSF5l2DcQuKxs0C7YH61W3ipvtZ&s=17&f=Guitar-Pro-8-3-3-Crack-With-License-Key-2023-Free-Download>

## 线索分析：LummaC2 V4.0 ?

### 病毒样本分析结果

<https://www.virustotal.com/gui/file/6ea22a21e9815986454f3b9c3dbdea9df158aad847b0620263e7524ea0cef5ba/behavior>

<https://www.virustotal.com/gui/file/8735ce3942864b8031fe311bdb1e77ece23e9f6794953a038484cee3e185bc09/behavior>

<https://www.joesandbox.com/analysis/1293164/0/html>

![Untitled](4.png)

**被删除的邮件时间在1min内，且没有用的邮件处于未读状态：提示规模化的自动化利用**

**LummaC Stealer样本**

木马本体编写：俄罗斯（MaaS）

木马外包装：中国（数字签名）

木马使用者：乌克兰-文尼察Oblast

木马的C2网站注册于：2023-10-11

木马下载于：2023-1015

样本首次被分析：2023-10-14

发作日期：至少2023-11-4以前

<https://twitter.com/anyrun_app/status/1698643550567579991>

这篇推文指出在其C2服务器的`/c2conf` 可以查看其配置。**注意：该木马可以执行携带的payload! 例如EXE, PowerShell**

**我注意到本机PowerShell的安全等级被设置为”Unrestricted”, 并不是默认的等级危险!!!**

此外木马似乎还请求了另一恶意软件(伪装成`clip.exe`)：<https://bazaar.abuse.ch/sample/fe3f04adc1fb9922ee259a9f355a79bb8cfac741d3490b4372cb80fe287877b1/>

<https://app.any.run/tasks/c1bd75b1-c7fd-4d57-86cd-77161313a399/>

提示**RaccoonStealer， 但也有可能是Amadey：**

<https://app.any.run/tasks/8d233d7c-bdc5-4feb-b02b-a5fa1e32f1cf/>

我怀疑这个被检测出虚拟环境了直接crash了。

相比之下是比较粗糙的一个。

本事件中没有执行恶意指令似乎，也许是C2没有下发命令。AnyRun这个在10-25才有的记录

没有网络连接，但是设置计划任务以实现持久化(远程控制？)。

### 对LummaC Stealer已有的分析

<https://www.esentire.com/blog/the-case-of-lummac2-v4-0> （最贴合的）

<https://ja.darktrace.com/blog/the-rise-of-the-lumma-info-stealer>

<https://blogs.vmware.com/security/2023/10/an-ilummanation-on-lummastealer.html>

<https://twitter.com/anyrun_app/status/1698643550567579991>

<https://asec.ahnlab.com/en/50594/> (还真是主要靠虚假的破解软件传播)

### 利用脚本分析

<https://github.com/esThreatIntelligence/RussianPanda_tools/blob/main/lummac2_config_extractor_build.py>

<https://github.com/esThreatIntelligence/RussianPanda/blob/main/lummac2_fetch_config_from_C2.py>

没有找到。这可能证实其使用C2服务器下发配置而非像老版本嵌入。

但是在Finsin的网站中找到了类似的URL，

包含

[http://[REDACTED].fun](http://[REDACTED].fun/)

[http://[REDACTED].pw](http://[REDACTED].pw/)

等。

这些Host似乎出自同一主体。好消息：**只要找到其中一个正在工作的服务器，我们至少可以知道对方做了什么！**

我们尝试发包：

![Untitled](5.png)

![Untitled](6.png)

![Untitled](7.png)

![Untitled](8.png)

![Untitled](9.png)

本次的样本对应C2服务器: [REDACTED].fun

`/api` `/c2conf` `/c2sock`

### 沙盒分析

<https://app.any.run/tasks/e66e2b32-73ae-4962-95e2-98c143fba6e4>

没有什么收获。因为C&C Server Down了

C2配置：（见文末）

可以看出目标有：

- 用户目录下的txt文件
- 用户目录下文件名含有`key` 的文件
- 用户目录下可能是加密货币敏感信息的相关文件
- 浏览器中加密货币的插件**（确认泄露）**
- **两步验证软件数据**
- 浏览器数据、**自动填充和密码管理器中的数据（确认泄露）**
- 浏览器**所有历史记录（确认泄露）**
- **Steam账户敏感数据（确认泄露）**
- **Telegram账号敏感数据（确认泄露）**
- 环境信息：进程、系统、硬件、已安装软件**（确认泄露）**

事实上还可能包括：

- 文档目录中的图片
- 注入正在运行的进程实现可持久化

但就这次来看，攻击者选择捞一波就结束。

窃取的信息以`.zip` 格式上传，内容示例：

![安装的软件](10.png)

安装的软件

![进程，最后两个即为病毒](11.png)

进程，最后两个即为病毒

![系统信息与Lumma标识符](12.png)

系统信息与Lumma标识符

![浏览器敏感信息](13.png)

浏览器敏感信息

![密码，自动填充信息，历史记录，网页数据，Cookie](14.png)

密码，自动填充信息，历史记录，网页数据，Cookie

![浏览器日志](15.png)

浏览器日志

推论：该木马植入后门后，进行规模化的自动化利用，且主要用于获取直接经济利益。

比较遗憾的一点是当时没及时查看c2conf具体的配置，不清楚具体的目标。也没想去any.run上跑一下看看。

其实该配置文件开头是**32-byte的XOR密钥，CyberChef就能秒掉**

搜索该C2服务器，在一个安全研究组织的网站上找到了唯一的结果：

<https://ioc.finsin.cl/Output_FINSIN_URL>

搜索`/c2conf` `/api`可以看到并非孤例，注册者均类似，怀疑出自同一主体。

但是，本次样本是`/api` ，~~不清楚为何~~(更换了新的Endpoint)。

<https://twitter.com/g0njxa/status/1702444978503360989>

考虑到这是MaaS且攻击者表现出了很低的专业性（未清空已删除邮件），有理由怀疑使用的是木马发布者的预设。

## 措施

重置密码管理器中的**所有**密码（有效的约30个）

网络账号登出其他设备

废弃所有私钥（3个）

更改邮箱密码

Steam两步验证（这次只有它受伤，因为只有Valve特立独行搞个自己的两步验证然后验证码收不到）

禁用PowerShell执行，破坏PowerShell用于利用的库

检查注册表与计划任务相关项（尤其是在沙盒中有可疑修改迹象的对象、Entry）

禁止IoC中的域名、IP等

清除浏览器缓存与Cookie

删除`%TEMP%`目录下的所有.exe

删除`%USERPROFILE%` `%APPDATA%` `%LOCALAPPDATA%`对应的在沙箱中被读写的文件

“快去叫360老将来！”

## 总结

对网络安全心存敬畏！注意保护好自己的硬件

> Punch line: 中国组织用俄罗斯的木马包装了一个钓鱼安装包卖给乌克兰人在冰岛网站上钓鱼并且在靶机上放了个后门与美国服务器联通

这次非常侥幸！电脑娱乐用，不和干活机子混用所以没有敏感信息；其上没有两步验证软件或者虚拟货币钱包；绝大部分的后渗透也被两步验证拦下了。

至少目前看来，C2服务器被Cloudflare拿下，也确实Down了。沙盒里没有展现其他的C2服务器连接。除非安置了别的后门，否则应该是安全的。

信息泄露是不可逆的，惨痛教训+1

几乎一致的样本（连压缩包都是一个名字）：<https://app.any.run/tasks/8d233d7c-bdc5-4feb-b02b-a5fa1e32f1cf/#>

- 不过这个似乎被木马认出来在虚拟环境里，自己crash掉了。

完全吻合的分析：<https://www.esentire.com/blog/the-case-of-lummac2-v4-0>

技术分析：<https://www.intrinsec.com/wp-content/uploads/2023/10/TLP-CLEAR-Lumma-Stealer-EN-Information-report.pdf>

已经可以说实锤了。

顺带一提Any.Run真是好东西，学校邮箱可以注册。

- 疑似同一攻击者的C2 配置，看得出来完全就是预设

    ```json
    {
        "v": 1,
        "c": [
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*.txt",
                "z": "Important Files/Profile",
                "d": 1
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*key*",
                "z": "Important Files/Profile",
                "d": 1
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*bitcoin*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*binance*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*exodus*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*coinbase*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*wallet*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*seed*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*pass*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*ledger*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*trezor*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*metamask*",
                "z": "Important Files/Profile",
                "d": 3
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*crypto*",
                "z": "Important Files/Profile",
                "d": 1
            },
            {
                "t": 0,
                "p": "%appdata%\\Binance",
                "m": "app-store.json",
                "z": "Wallets/Binance",
                "d": 1
            },
            {
                "t": 0,
                "p": "%appdata%\\Binance",
                "m": ".finger-print.fp",
                "z": "Wallets/Binance",
                "d": 1
            },
            {
                "t": 0,
                "p": "%appdata%\\Binance",
                "m": "simple-storage.json",
                "z": "Wallets/Binance",
                "d": 1
            },
            {
                "t": 0,
                "p": "%appdata%\\Electrum\\wallets",
                "m": "*",
                "z": "Wallets/Electrum",
                "d": 1
            },
            {
                "t": 0,
                "p": "%appdata%\\Ethereum",
                "m": "keystore",
                "z": "Wallets/Ethereum",
                "d": 1
            },
            {
                "t": 0,
                "p": "%appdata%\\Exodus\\exodus.wallet",
                "m": "*",
                "z": "Wallets/Exodus",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\Ledger Live",
                "m": "*",
                "z": "Wallets/Ledger Live",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\atomic\\Local Storage\\leveldb",
                "m": "*",
                "z": "Wallets/Atomic",
                "d": 2
            },
            {
                "t": 0,
                "p": "%localappdata%\\Coinomi\\Coinomi\\wallets",
                "m": "*",
                "z": "Wallets/Coinomi",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\Authy Desktop\\Local Storage\\leveldb",
                "m": "*",
                "z": "Wallets/Authy Desktop",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\Bitcoin\\wallets",
                "m": "*",
                "z": "Wallets/Bitcoin core",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\com.liberty.jaxx\\IndexedDB",
                "m": "*.leveldb",
                "z": "Wallets/JAXX New Version",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\Electrum\\wallets",
                "m": "*",
                "z": "Wallets/Electrum",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\AnyDesk",
                "m": "*.conf",
                "z": "Applications/AnyDesk",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\FileZilla",
                "m": "recentservers.xml",
                "z": "Applications/FileZilla",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\FileZilla",
                "m": "sitemanager.xml",
                "z": "Applications/FileZilla",
                "d": 2
            },
            {
                "t": 0,
                "p": "%userprofile%",
                "m": "*.kbdx",
                "z": "Applications/KeePass",
                "d": 2
            },
            {
                "t": 0,
                "p": "%programfiles%\\Steam",
                "m": "ssfn*",
                "z": "Applications/Steam",
                "d": 2
            },
            {
                "t": 0,
                "p": "%programfiles%\\Steam\\config",
                "m": "*",
                "z": "Applications/Steam/config",
                "d": 2
            },
            {
                "t": 0,
                "p": "%appdata%\\Telegram Desktop",
                "m": "*s",
                "z": "Applications/Telegram",
                "d": 2
            },
            {
                "t": 1,
                "e": [
                    {
                        "en": "ejbalbakoplchlghecdalmeeeajnimhm",
                        "ez": "MetaMask"
                    },
                    {
                        "en": "nkbihfbeogaeaoehlefnkodbefgpgknn",
                        "ez": "MetaMask"
                    },
                    {
                        "en": "egjidjbpglichdcondbcbdnbeeppgdph",
                        "ez": "Trust Wallet"
                    },
                    {
                        "en": "ibnejdfjmmkpcnlpebklmnkoeoihofec",
                        "ez": "TronLink"
                    },
                    {
                        "en": "fnjhmkhhmkbjkkabndcnnogagogbneec",
                        "ez": "Ronin Wallet"
                    },
                    {
                        "en": "fhbohimaelbohpjbbldcngcnapndodjp",
                        "ez": "Binance Chain Wallet"
                    },
                    {
                        "en": "ffnbelfdoeiohenkjibnmadjiehjhajb",
                        "ez": "Yoroi"
                    },
                    {
                        "en": "jbdaocneiiinmjbjlgalhcelgbejmnid",
                        "ez": "Nifty"
                    },
                    {
                        "en": "afbcbjpbpfadlkmhmclhkeeodmamcflc",
                        "ez": "Math"
                    },
                    {
                        "en": "hnfanknocfeofbddgcijnmhnfnkdnaad",
                        "ez": "Coinbase"
                    },
                    {
                        "en": "hpglfhgfnhbgpjdenjgmdgoeiappafln",
                        "ez": "Guarda"
                    },
                    {
                        "en": "blnieiiffboillknjnepogjhkgnoapac",
                        "ez": "EQUA"
                    },
                    {
                        "en": "cjelfplplebdjjenllpjcblmjkfcffne",
                        "ez": "Jaxx Liberty"
                    },
                    {
                        "en": "fihkakfobkmkjojpchpfgcmhfjnmnfpi",
                        "ez": "BitApp"
                    },
                    {
                        "en": "kncchdigobghenbbaddojjnnaogfppfj",
                        "ez": "iWlt"
                    },
                    {
                        "en": "kkpllkodjeloidieedojogacfhpaihoh",
                        "ez": "EnKrypt"
                    },
                    {
                        "en": "amkmjjmmflddogmhpjloimipbofnfjih",
                        "ez": "Wombat"
                    },
                    {
                        "en": "nlbmnnijcnlegkjjpcfjclmcfggfefdm",
                        "ez": "MEW CX"
                    },
                    {
                        "en": "nanjmdknhkinifnkgdcggcfnhdaammmj",
                        "ez": "Guild"
                    },
                    {
                        "en": "nkddgncdjgjfcddamfgcmfnlhccnimig",
                        "ez": "Saturn"
                    },
                    {
                        "en": "cphhlgmgameodnhkjdmkpanlelnlohao",
                        "ez": "NeoLine"
                    },
                    {
                        "en": "nhnkbkgjikgcigadomkphalanndcapjk",
                        "ez": "Clover"
                    },
                    {
                        "en": "kpfopkelmapcoipemfendmdcghnegimn",
                        "ez": "Liquality"
                    },
                    {
                        "en": "aiifbnbfobpmeekipheeijimdpnlpgpp",
                        "ez": "Terra Station"
                    },
                    {
                        "en": "dmkamcknogkgcdfhhbddcghachkejeap",
                        "ez": "Keplr"
                    },
                    {
                        "en": "fhmfendgdocmcbmfikdcogofphimnkno",
                        "ez": "Sollet"
                    },
                    {
                        "en": "cnmamaachppnkjgnildpdmkaakejnhae",
                        "ez": "Auro"
                    },
                    {
                        "en": "jojhfeoedkpkglbfimdfabpdfjaoolaf",
                        "ez": "Polymesh"
                    },
                    {
                        "en": "flpiciilemghbmfalicajoolhkkenfe",
                        "ez": "ICONex"
                    },
                    {
                        "en": "nknhiehlklippafakaeklbeglecifhad",
                        "ez": "Nabox"
                    },
                    {
                        "en": "hcflpincpppdclinealmandijcmnkbgn",
                        "ez": "KHC"
                    },
                    {
                        "en": "ookjlbkiijinhpmnjffcofjonbfbgaoc",
                        "ez": "Temple"
                    },
                    {
                        "en": "mnfifefkajgofkcjkemidiaecocnkjeh",
                        "ez": "TezBox"
                    },
                    {
                        "en": "lodccjjbdhfakaekdiahmedfbieldgik",
                        "ez": "DAppPlay"
                    },
                    {
                        "en": "ijmpgkjfkbfhoebgogflfebnmejmfbm",
                        "ez": "BitClip"
                    },
                    {
                        "en": "lkcjlnjfpbikmcmbachjpdbijejflpcm",
                        "ez": "Steem Keychain"
                    },
                    {
                        "en": "onofpnbbkehpmmoabgpcpmigafmmnjh",
                        "ez": "Nash Extension"
                    },
                    {
                        "en": "bcopgchhojmggmffilplmbdicgaihlkp",
                        "ez": "Hycon Lite Client"
                    },
                    {
                        "en": "klnaejjgbibmhlephnhpmaofohgkpgkd",
                        "ez": "ZilPay"
                    },
                    {
                        "en": "aeachknmefphepccionboohckonoeemg",
                        "ez": "Coin98"
                    },
                    {
                        "en": "bhghoamapcdpbohphigoooaddinpkbai",
                        "ez": "Authenticator"
                    },
                    {
                        "en": "dkdedlpgdmmkkfjabffeganieamfklkm",
                        "ez": "Cyano"
                    },
                    {
                        "en": "nlgbhdfgdhgbiamfdfmbikcdghidoadd",
                        "ez": "Byone"
                    },
                    {
                        "en": "infeboajgfhgbjpjbeppbkgnabfdkdaf",
                        "ez": "OneKey"
                    },
                    {
                        "en": "cihmoadaighcejopammfbmddcmdekcje",
                        "ez": "Leaf"
                    },
                    {
                        "en": "gaedmjdfmmahhbjefcbgaolhhanlaolb",
                        "ez": "Authy"
                    },
                    {
                        "en": "oeljdldpnmdbchonielidgobddfffla",
                        "ez": "EOS Authenticator"
                    },
                    {
                        "en": "ilgcnhelpchnceeipipijaljkblbcob",
                        "ez": "GAuth Authenticator"
                    },
                    {
                        "en": "imloifkgjagghnncjkhggdhalmcnfklk",
                        "ez": "Trezor Password Manager"
                    },
                    {
                        "en": "bfnaelmomeimhlpmgjnjophhpkkoljpa",
                        "ez": "Phantom"
                    },
                    {
                        "en": "ppbibelpcjmhbdihakflkdcoccbgbkpo",
                        "ez": "UniSat"
                    }
                ],
                "n": [
                    {
                        "p": "%localappdata%\\Google\\Chrome\\User Data",
                        "z": "Chrome"
                    },
                    {
                        "p": "%localappdata%\\Chromium\\User Data",
                        "z": "Chromium"
                    },
                    {
                        "p": "%localappdata%\\Microsoft\\Edge\\User Data",
                        "z": "Edge"
                    },
                    {
                        "p": "%localappdata%\\Kometa\\User Data",
                        "z": "Kometa"
                    },
                    {
                        "p": "%appdata%\\Opera Software\\Opera Stable",
                        "z": "Opera Stable"
                    },
                    {
                        "p": "%appdata%\\Opera Software\\Opera GX Stable",
                        "z": "Opera GX Stable"
                    },
                    {
                        "p": "%appdata%\\Opera Software\\Opera Neon\\User Data",
                        "z": "Opera Neon"
                    },
                    {
                        "p": "%localappdata%\\BraveSoftware\\Brave-Browser\\User Data",
                        "z": "Brave Software"
                    },
                    {
                        "p": "%localappdata%\\Comodo\\Dragon\\User Data",
                        "z": "Comodo"
                    },
                    {
                        "p": "%localappdata%\\CocCoc\\Browser\\User Data",
                        "z": "CocCoc"
                    }
                ]
            },
            {
                "t": 2,
                "p": "%appdata%\\Mozilla\\Firefox\\Profiles",
                "z": "Mozilla Firefox"
            }
        ]
    }
    ```
