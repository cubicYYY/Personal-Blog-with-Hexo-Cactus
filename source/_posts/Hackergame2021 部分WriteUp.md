---
title: Hackergame2021 部分WriteUp
date: 2021-10-30 16:11
tags:
- Hackergame
- Write-Up
categories: 
- CTF
---

# WriteUp by YYY

------

## 一点碎碎念  

这是我的第一次Hackergame/CTF比赛，感觉很有意思。前几天心态有点崩，好在调整过来了。大家还是要把Hackergame当做game。最开心的是认识了好多大佬，抱大腿.jpg  

![emoji](https://img2020.cnblogs.com/blog/1335480/202110/1335480-20211030161203937-1693462675.png)

对自己的吐槽：EasyRSA差点做出来，扩展欧几里得写错弃疗了；RAID0卡在软件没有激活码不让保存；和各种小测撞；各种体调不良，饮食睡眠不佳；晚上学校断电没法做，我又是大夜猫子……奇奇怪怪挡路的东西一直不少，归根结底还是太菜了啦。  
总之非常感谢@TonyCrane大佬、GodSpeed大佬及群内各成员的鼓励支持帮助（膜不算）。
希望我能早日学会binary和math。  

------

## 签到  

这题二分一下page参数即可。  
~~鼠标连点器~~  

------

## 进制十六——参上  

使用Hex Editor Neo直接新建文件抄写即可。当然OCR识别也行吧。

------

## 去吧！追寻自由的电波  

下载音频后可以听出是一段人声，但是语速极快。于是使用Adobe Audition CC打开，效果->伸缩与变调放慢速度，适当调节音高就能开始听写了。可以发现这是**NATO Phonetic Alphabet**：
>A ALPHA
>B BRAVO
>C CHARLIE
>D DELTA
>E ECHO
>F FOXTROT
>G GOLF
>H HOTEL
>I INDIA
>J JULIET
>K KILO
>L LIMA
>M MIKE
>N NOVEMBER
>O OSCAR
>P PAPA
>Q QUEBEC
>R ROMEO
>S SIERRA
>T TANGO
>U UNIFORM
>V VICTOR
>W WHISKEY
>X X-RAY
>Y YANKEE
>Z ZULU  

------

## 猫咪问答 Pro Max  

General× 杂技√  

1. [经典WebArchive](http://web.archive.org/web/20181004003308/http://sec.ustc.edu.cn/doku.php/codes) 第一行就有答案:20150504

2. [https://lug.ustc.edu.cn/wiki/intro/](https://lug.ustc.edu.cn/wiki/intro/)  
    >
    > 此处资料显示是4次，但并非最新，我后来手动尝试才得知是5

3. [FTP服务器](https://ftp.lug.ustc.edu.cn/%E6%B4%BB%E5%8A%A8/2016.06.16_%E6%B4%BB%E5%8A%A8%E5%AE%A4%E6%90%AC%E8%BF%81/IMG_20160616_133655.jpg) Obviously，答案是Development Team of Library ~~后来得知网站首页news头图就有，我还找了好久~~

4. 谷歌关键词`SIGBOVIK2021` `Newcomb-Benford`直接就能找到原论文[http://sigbovik.org/2021/proceedings.pdf](http://sigbovik.org/2021/proceedings.pdf) 直接`Ctrl+F`找到文章，发现有14个Figures，排除第一个在Background里的Figure后得到答案为13

5. 谷歌关键词`protocol` `report` 找到[https://www.rfc-editor.org/rfc/rfc8962.html](https://www.rfc-editor.org/rfc/rfc8962.html)  

> 6.Reporting Offenses
Send all your reports of possible violations and all tips about wrongdoing to **/dev/null**. The Protocol Police are listening and will take care of it.

~~还挺幽默，一开始还真以为有police在listening~~

------

## 卖瓜  

20不是3的倍数，乍看似乎不可能用6和9加和得到。但随便试了试，发现9斤的瓜很多很多时会溢出为-9223372036854775808，据此判断为int64溢出，判断应当在此处加以利用。  
我们需要让这个数字稍微溢出一点，不能溢出太多。也就是略大于floor(9223372036854775808/9)，使得溢出为-9223372036854775808以外的数字，并且让该数字到20的距离能被3整除。之后直接加回20就行（简单小学(?)算数，屡有即将做出来时加过头超过20的惨剧hhh）
灵感来源：[CTF 两道web整数溢出题目(猫咪银行和ltshop)](https://www.bbsmax.com/A/pRdByjq65n/)  

------

## 透明的文件  

根据题面和文件会发现这是终端的颜色代码，网络搜索终端颜色代码格式后将所有`[`前补上一个`\e`，然后`echo -e "{内容}"`就行啦。记得执行前把终端先用字符填满，不然可能画不完整。  
![chars](https://img2020.cnblogs.com/blog/1335480/202110/1335480-20211030154901529-149476643.jpg)

------

## 旅行照片

~~简简单单开个盒~~
旅游，海滩，汉字，说明这是国内一个海边旅游景点。  
蓝色KFC？这可不常见，应该有不少人打卡了吧。
百度搜索`蓝色KFC`，第一项就是某红书的`秦皇岛蒂凡尼蓝秦皇岛网红打卡|国内唯一蒂芙尼蓝色肯德基`
百度地图定位发现这家店是某海底世界分店，直接得到电话号码。  
[https://www.earthol.org/](https://www.earthol.org/)上通过街景发现三个汉字“海|豚|馆”  
![building](https://img2020.cnblogs.com/blog/1335480/202110/1335480-20211030154657507-2019993326.jpg)

对照地图，视线和海岸线大约成45°，推测应当在东南方向。进而发现影子西斜，应当在下午/傍晚  
通过绘制各个水平线找出灭点可以推测楼层（知乎有些文章有详细说明）。我的方法是找到对面楼“最矩形”变形最少的窗户判断为同一楼层，然后数，发现是13层左右（经尝试发现是14层）

------

## FLAG助力大红包  

既然是和ip有关，第一时间想到的就是`X-Forwarded-For`，果然成功了。因为每个`/8`ip段（也就是例如255.0.0.0-255.255.255.255）都只能算一次，我们使用BurpSuite的Intruder，将Post数据中的ip和`X-Forwarded-For`的ip首段打上标记，选择`Battering Ram`模式（让两处参数一致），构造0\~255的字典，自动化访问`0.114.114.114`、`1.114.114.114`、`2.114.114.114`……`255.114.114.114`达成刚好256个助力获得flag。由于每次间隔2s，2s\*256=512s，小于题目时长限制10min=600s所以是可行的。

------

## Amnesia-轻度失忆  

直接`putchar()`逐个打印绕过即可。

------

## 图之上的信息

GraphQL的利用。访问`/graphql?query={}`发现存在利用可能。查阅[资料](https://mp.weixin.qq.com/s/gp2jGrLPllsh5xn7vn9BwQ)后得知可以利用内省注入。没有UI界面我直接地址栏输入。换行替换为`%0A`后，访问`/graphql?query={__schema%0A{types%0A{name}}}`得到：

```json
{"data":{"__schema":{"types":[{"name":"Query"},{"name":"GNote"},{"name":"Int"},{"name":"String"},{"name":"GUser"},{"name":"Boolean"},{"name":"__Schema"},{"name":"__Type"},{"name":"__TypeKind"},{"name":"__Field"},{"name":"__InputValue"},{"name":"__EnumValue"},{"name":"__Directive"},{"name":"__DirectiveLocation"}]}}}
```

发现有个`GUser`类型。接下来访问`/graphql?query={__type(name:"GUser"){name%0Afields{name%0Atype{name,kind}}}}`爆出字段名`privateEmail`，然后直接`/graphql?query={user(id:1){id,privateEmail}}`得到flag\.

------

## 赛博厨房

- Level0：直接写
- Level1：复制粘贴发现有长度限制 goto优化行数
 然后不会了。

------

## 阵列恢复大师-RAID5

直接丢进RAID Reconstructor 5里面跑得到镜像文件。Windows上并没法直接读，于是丢进Diskinternals Linux Reader里读文件导出。执行getflag.py却提示`Did you recover my data correctly?`，疑惑之下换WSL(Kali Linux)执行就成功了。
~~吐槽：RAID0的XFS需要注册码没法搞。WSL也mount不上，看来还是虚拟机靠谱。~~

------

## 助记词-第一顿大餐

代码审计后发现目的是延迟尽可能高。用BurpSuite改包在POST数据里复制出来很多行注记词提交就有了flag1。就像这样：  

```json
[
    "necessary science growth addition",
    "necessary science growth addition",
    （重复好多好多次）
    "necessary science growth addition"
]
```

如果不行CLEAR再试试。  
![clear it](https://img2020.cnblogs.com/blog/1335480/202110/1335480-20211030155443141-1922071122.jpg)

小坑：token要跟上才能拿到flag，不过有时自动获取的token无效，使得flag只显示为true，因为加号没有URLEncode转义，不知道是不是bug.  

------

## 马赛克

模拟题。大概思路是先扫一遍，把能确定的先确定下来。之后再dfs剩下的块（不需要全部求出，毕竟这题的二维码纠错拉满了）。
代码很丑对吧QAQ（当时不熟悉ndarray操作，全部当做list做一遍）  

```python
from PIL import Image
import math
import numpy as np
# import scipy
import matplotlib.pyplot as plt

SIZE = 627
MSBLOCK = 23
QRBLOCK = 11
X, Y = 103, 137
def ImageToMatrix(filename):
    # 读取图片
    im = Image.open(filename)
    # 显示图片
#     im.show()  
    width,height = im.size
    im = im.convert("L") 
    data = im.getdata()
    data = np.matrix(data,dtype='float')/255.0
    new_data = np.reshape(data,(width,height))
    return new_data
#     new_im = Image.fromarray(new_data)
#     # 显示图片
#     new_im.show()
def MatrixToImage(data):
    data = data*255
    new_im = Image.fromarray(data.astype(np.uint8))
    return new_im
    
pre_arr =  [[0 for col in range(SIZE)] for row in range(SIZE)]
res_arr =  [[1 for col in range(SIZE//QRBLOCK)] for row in range(SIZE//QRBLOCK)]
kimeta =  [[0 for col in range(SIZE//QRBLOCK)] for row in range(SIZE//QRBLOCK)]#钦定了
EACH = int(math.ceil(MSBLOCK/QRBLOCK)) #EACH=3
filename = 'pixelated_qrcode.bmp'
data = ImageToMatrix(filename)
np.set_printoptions(threshold=1145141919810)
for i in range(0,SIZE):
    for j in range(0,SIZE):
            pre_arr[i][j]=data[i].tolist()[0][j]
            if i%QRBLOCK==0 and j%QRBLOCK==0:
                res_arr[i//QRBLOCK][j//QRBLOCK]=int(data.tolist()[i][j])
# print(pre_arr)
def  check(i,j,now):
    if i<=9 or i>=51 or j<=12 or j>=54:
        if not (pre_arr[i*QRBLOCK+3][j*QRBLOCK+3] == now):#随便写的offset
            return False
    if (kimeta[i][j] or vis[i][j]) and res_arr[i][j]!=now:
        return False
    return True


def getRange(i,j):
    return i*QRBLOCK,j*QRBLOCK,(i+1)*QRBLOCK-1,(j+1)*QRBLOCK-1 #x1,y1,x2,y2

vis =  [[0 for col in range(SIZE//QRBLOCK)] for row in range(SIZE//QRBLOCK)]
def calcDelta(Mi,Mj,fillN,comp,LUR,LUC):
    fn = fillN
    for xx in range(LUR,LUR+EACH) :
        for yy in range(LUC,LUC+EACH):
            if not check(xx,yy,fn&1):
                return 114514
            fn = fn>>1
    avr = 0
    qx1,qy1,qx2,qy2 = X+Mi*MSBLOCK,Y+Mj*MSBLOCK,X+(Mi+1)*MSBLOCK-1,Y+(Mj+1)*MSBLOCK-1
    for x in range(LUR,LUR+EACH) :
        for y in range(LUC,LUC+EACH):
            if (fillN&1):
                x1,y1,x2,y2 = getRange(x,y)
                inx1,iny1,inx2,iny2 = max(x1,qx1),max(y1,qy1),min(x2,qx2),min(y2,qy2)
                avr = avr + (iny2-iny1+1)*(inx2-inx1+1)
            fillN = fillN >> 1
    newres = avr/(MSBLOCK*MSBLOCK)
    if (newres < comp-0.1):
        return 1919810
    return abs(math.floor((avr/(MSBLOCK*MSBLOCK))*255)/255-comp)

def putRec(i,j):#放置识别码块
    kimeta[i][j]=1
    res_arr[i][j]=0

    kimeta[i][j+1]=1
    res_arr[i][j+1]=1
    kimeta[i][j-1]=1
    res_arr[i][j-1]=1
    kimeta[i+1][j]=1
    res_arr[i+1][j]=1
    kimeta[i-1][j]=1
    res_arr[i-1][j]=1
    kimeta[i+1][j+1]=1
    res_arr[i+1][j+1]=1
    kimeta[i-1][j-1]=1
    res_arr[i-1][j-1]=1
    kimeta[i+1][j-1]=1
    res_arr[i+1][j-1]=1
    kimeta[i-1][j+1]=1
    res_arr[i-1][j+1]=1

    for x in range(i-2,i+3):
        for y in range(j-2,j+3):
            if abs(x-i)==2 or abs(y-j)==2:
                kimeta[x][y]=1
                res_arr[x][y]=0

putRec(28,28)
putRec(28,50)
putRec(50,50)
putRec(50,28)

for i in range(9,52):
    for j in range(12,55):
        if i==9 or i==51 or j==12 or j==54:
            kimeta[i][j]=1
            res_arr[i][j]=pre_arr[i*QRBLOCK+3][j*QRBLOCK+3]

new_arr =  [[1 for col in range(SIZE)] for row in range(SIZE)]
fked =  [[0 for col in range(20)] for row in range(20)]

cntt = 0
savecnt=0
def dfs(i,j):#从马赛克块的i行j列开始
    global cntt,savecnt
    cntt = cntt+1
    LUR , LUC= (X+i*MSBLOCK)//QRBLOCK , (Y+j*MSBLOCK)//QRBLOCK
    #落在哪个二维码方块上
    if (cntt==200000):#保存中途结果
        for i in range(0,SIZE):
            for j in range(0,SIZE):
                    new_arr[i][j]=res_arr[i//QRBLOCK][j//QRBLOCK]
        MatrixToImage(np.array(new_arr)).save('mid%s.bmp'%savecnt)
        savecnt = savecnt + 1
        cntt = 0
    if i<0 or i>=20 or j<0 or j>=20 or fked[i][j]:
        return False
    fked[i][j]=1
    
    nowMin , nowSol = 114514191981.0 , (1<<(EACH*EACH))-1
    tmpp =  [[0 for col in range(3)] for row in range(3)]
    for x in range(LUR,LUR+EACH) :
        for y in range(LUC,LUC+EACH):
            tmpp[x-LUR][y-LUC]=vis[x][y]
    for filN in range(0,1<<(EACH*EACH)):#枚举每个马赛克块影响到的3*3=9个QR块
        tmp = calcDelta(i, j, filN, pre_arr[X+i*MSBLOCK+2][Y+j*MSBLOCK+2],LUR,LUC)#同样是乱写的offset+2
        if tmp < 0.00000001:
            nowSol = filN
            for x in range(LUR,LUR+EACH) :
                for y in range(LUC,LUC+EACH):
                    vis[x][y] = 1
                    res_arr[x][y]=nowSol&1
                    nowSol = nowSol>>1
            if ((i+1<20 and (not fked[i+1][j]) and dfs(i+1,j)) or (j+1<20 and (not fked[i][j+1]) and dfs(i,j+1)) or (i-1>=0 and (not fked[i-1][j]) and dfs(i-1,j)) or (j-1>=0 and (not fked[i][j-1]) and dfs(i,j-1))):
                fked[i][j]=0
                for x in range(LUR,LUR+EACH) :
                    for y in range(LUC,LUC+EACH):
                        vis[x][y] = tmpp[x-LUR][y-LUC]
                return True
            
    fked[i][j]=0
    for x in range(LUR,LUR+EACH) :
        for y in range(LUC,LUC+EACH):
            vis[x][y] = tmpp[x-LUR][y-LUC]
    return False
    

#预先扫描，把能确定的、没有多解的先填上
for i in range(0,20):
    for j in range(0,20):
        LUR , LUC= (X+i*MSBLOCK)//QRBLOCK , (Y+j*MSBLOCK)//QRBLOCK
        succnt = 0
        for filN in range(0,1<<(EACH*EACH)):
                tmp = calcDelta(i, j, filN, pre_arr[X+i*MSBLOCK+2][Y+j*MSBLOCK+2],LUR,LUC)
                if tmp < 0.00000001:
                    succnt = succnt + 1
                    nowSol = filN
        if succnt == 1:
            for x in range(LUR,LUR+EACH) :
                for y in range(LUC,LUC+EACH):
                    kimeta[x][y] = 1
                    res_arr[x][y]=nowSol&1
                    nowSol = nowSol>>1
dfs(5,5)

for i in range(0,SIZE):
    for j in range(0,SIZE):
        new_arr[i][j]=res_arr[i//QRBLOCK][j//QRBLOCK]
MatrixToImage(np.array(new_arr)).save('resu.bmp')
```

最终效果：![QRCode](https://img2020.cnblogs.com/blog/1335480/202110/1335480-20211030160204036-1937509495.jpg)

------

## minecRaft

web× reverse√  
打开网页查看js，找到flag.js，在[jsnice.org](http://jsnice.org/)反混淆后自己手动再替换下几个迷人眼的常量，之后进行代码审计，会发现这是个TEA加密。如何发现是TEA呢？我搜了好久，后来有人告诉我只需要搜常量就行（还需要学习一个人生经验）。  
我们需要找到一个字符串s，使得s.encrypt("{那串数字密钥}")=== "6f......0c"  
把密文切片避免转换后数字过大，在题目页面下`F12`，进入Console里执行：

```javascript
window.btoa(LongToStr4(Base16ToLong('6fbde674'))+LongToStr4(Base16ToLong('819a59bf'))+LongToStr4(Base16ToLong('a1209256'))+LongToStr4(Base16ToLong('5b4ca2a7'))+LongToStr4(Base16ToLong('a11dc670'))+LongToStr4(Base16ToLong('c678681d'))+LongToStr4(Base16ToLong('af4afb67'))+LongToStr4(Base16ToLong('04b82f0c')))
```  

得到密文dOa9b79ZmoFWkiChp6JMW3DGHaEdaHjGZ/tKrwwvuAQ=  
谷歌`TEA decryption online`，进入[https://www.mefancy.com/obfuscation/encryption-generator.php](https://www.mefancy.com/obfuscation/encryption-generator.php)把上面那串数字密钥（13...）还有密文丢进去就出flag了。

------

## p😭q

老外大奥流泪.gif  早知道，还是原道.jpg  
先把gif的帧提取出来方便处理：

```python
import os
import sys
from PIL import Image

def extractFrames(inGif, outFolder):
    frame = Image.open(inGif)
    nframes = 0
    while frame:
        frame.save('./dest/%s.png' % (nframes))
        nframes += 1
        try:
            frame.seek(nframes)
        except EOFError:
            break;
    return True

if __name__ == '__main__':
    image = os.path.abspath(sys.argv[1])
    dest = os.path.join(os.path.dirname(image), "dest")
    if not os.path.exists(dest):
        os.mkdir(dest)
    extractFrames(image, dest)
```  

然后原题给啥库就用啥库，逆回去：

```python
from array2gif import write_gif  # version: 1.0.4
import librosa  # version: 0.8.1
import numpy as np  # version: 1.19.5
import soundfile as sf
from PIL import Image
import matplotlib.pyplot as plt

def ImageToMatrix(filename):
    im = Image.open(filename)
    height,width = im.size
    im = im.convert('L')
    data = im.getdata()
    data = np.matrix(data,dtype='float')/255.0
    new_data = np.reshape(data,(width,height))
    return new_data
np.set_printoptions(threshold=1145141919810)
num_freqs = 32
quantize = 2
min_db = -60
max_db = 30
fft_window_size = 2048
frame_step_size = 512
window_function_type = 'hann'
red_pixel = [255, 0, 0]
white_pixel = [255, 255, 255]
sample_rate = 22050
lis =  [[0.0 for col in range(587)] for row in range(32)]
imgg = [[0.0 for col in range(130)] for row in range(92)]
for ii in range (0,587):
    filename = './dest/%s.png'%ii
    print(filename)
    data = ImageToMatrix(filename)
    for i in range(0,92):
        for j in range(0,130):
            imgg[i][j]=data.tolist()[i][j]
    for xpos in range(0,32):
        for scan in range(0,92):
            if imgg[scan][xpos*4+2]<1.0:  
                lis[xpos][ii]=float((91-scan)-60)
                break

spectrogram =  np.array(lis)
audio_signal = librosa.feature.inverse.mel_to_audio(librosa.db_to_power(spectrogram), sr=sample_rate, n_fft=fft_window_size*2, hop_length=frame_step_size, window=window_function_type)
sf.write('newnew.wav', audio_signal, sample_rate) 
```  

最后在Adobe Audition CC里随便处理处理，勉强能听了，开始努力听写：
`Theflagis f,l,a,g ......`  
