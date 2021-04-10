---
title: 自动化图片脚本
date: '2018-04-26'
---

为了方便记笔记而写的随时进行屏幕截图并/或去除白底色并上传图片到OSS服务的脚本。

<!-- more -->

最近写笔记的时候遇到需要把纸上手写/屏幕上的内容放到网页上的问题，这个是有一套手动的解决方案的：印象笔记文档扫描，保存同步到电脑，电脑上下载下来，PS去掉白色背景，上传到OSS，获取图片链接，插入到笔记上。不过整个过程太过冗杂，让我感到难受，就想办法写个脚本。原本想着全套包揽，但之后发现摄像头文档扫描和手机电脑文件同步这两点实在是不好做，就还是用印象笔记了。所以脚本做的事情是：可以截图，给文档图片去除白色背景（透明色，可选功能），上传到OSS，把链接地址复制到剪贴板。

## 屏幕截图

主要使用了github上一个现成的python库，用pyQt5写成，轻量级，好用。在源码的基础上修改了一下就能用了。

https://github.com/SeptemberHX/screenshot

## OSS API

先想办法以编程的方式上传到阿里云的OSS，翻了半天他们的API官方文档，后来发现有现成的Python SDK，何乐而不为：）

```bash
pip install oss2
```

就可以安装了。使用起来也很方便，用权限ID与密钥、终端名称等初始化，之后直接上传一个文件对象就行。专门给自动上传脚本设了文件前缀。不过本来就是拿来做图床的OSS结构也挺乱的。

上传完成之后用python的跨平台剪贴板包`pyperclip`来把地址复制到剪贴板

```python
import pyperclip

pyperclip.copy(address)
```

这样基础结构就完成了。

## 去除图片白底色

还有比较麻烦的一点是背景透明化，因为白色背景的图片放到别的颜色的网页上会很丑。印象笔记扫描模式出来的照片背景基本都是很白的，之前直接在PS里面选中颜色（容差也不大）删除就行。这一步还是可以编程实现的。

Python的图片处理库：PIL(Python Image Library)。步骤：读入文件、转换为RGBA模式、遍历像素，如果是白色就把Alpha通道设为0、保存。代码：

```python
from PIL import Image

im = Image.open(filename).convert('RGBA')
pix = im.load()
for y in range(im.size[1]):
    for x in range(im.size[0]):
        if pix[x,y][0] > 220 and pix[x,y][2] > 220 and pix[x,y][3] > 220:
            pix[x,y] = (255,255,255,0)
filename = os.path.splitext(filename)[0] + '_rmbg.png'
im.save(filename, 'PNG')
```

## 配置Path

这样基本就完成了。为了让脚本能在任何路径下访问到，新建了路径D:\scripts\，并把它加入到系统Path中，这样里面的脚本就可以直接执行了。目录下python文件和bat脚本，通过执行bat来调用python脚本。ossup.bat：

```bat
python D:\scripts\oss.py %1 %2
```

大功告成。

最后附上完整python文件：

```python
import os, sys, pyperclip
import oss2
from PIL import Image

if len(sys.argv) < 2:
    print('No enough arguments!')
    exit()

filename = os.path.basename(sys.argv[1])

if not os.path.exists(filename):
    print('File not exists : ' + filename)
    exit()

if 'bg' in sys.argv:
    print('Processing background...')
    im = Image.open(filename).convert('RGBA')
    pix = im.load()
    for y in range(im.size[1]):
        for x in range(im.size[0]):
            if pix[x,y][0] > 220 and pix[x,y][2] > 220 and pix[x,y][3] > 220:
                pix[x,y] = (255,255,255,0)
    filename = os.path.splitext(filename)[0] + '_rmbg.png'
    im.save(filename, 'PNG')

print('Initializing oss sdk...')

access_key_id = os.getenv('OSS_TEST_ACCESS_KEY_ID', '<>')
access_key_secret = os.getenv('OSS_TEST_ACCESS_KEY_SECRET', '<>')
bucket_name = os.getenv('OSS_TEST_BUCKET', '<>')
endpoint = os.getenv('OSS_TEST_ENDPOINT', '<>')

bucket = oss2.Bucket(oss2.Auth(access_key_id, access_key_secret), endpoint, bucket_name)

oss_name = 'autoup/' + filename

print('Filename : ' + oss_name + '\nUploading')

with open(filename, 'rb') as f:
    bucket.put_object(oss_name, f)

if 'bg' in sys.argv:
    os.remove(filename)

address = '<>' + oss_name
pyperclip.copy(address)

print('Address copied to clipboard!\nBye~')
```

效果图：

![example](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/ossup-example.png)