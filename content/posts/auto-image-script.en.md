---
title: Automated Image Script
date: '2018-04-26'
---

A script written at any time for the convenience of taking notes and/or removing white background and uploading images to the OSS service.

<!-- more -->

When I recently wrote notes, I encountered the problem of putting the handwritten/screen content on the paper on the webpage. This is a manual solution: scan the Evernote document, save it to the computer, download it on the computer, PS Remove the white background, upload it to OSS, get the image link, and insert it into the note. However, the whole process was too complicated and made me feel uncomfortable. I just tried to write a script. I originally thought about a full set of packages, but after discovering that the camera document scanning and the mobile computer file synchronization are really difficult to do, I still use Evernote. So what the script does is: you can take a screenshot, remove the white background (transparent color, optional function) from the document image, upload it to OSS, and copy the link address to the clipboard.

## Screenshots

Mainly used a ready-made python library on github, written in pyQt5, lightweight, easy to use. Modify it on the basis of the source code to use it.

https://github.com/SeptemberHX/screenshot

## OSS API

I first tried to upload it to Alibaba Cloud's OSS programmatically. I flipped through their API official documentation for a long time, and later found out that there is a ready-made Python SDK. Why not:)

```bash
pip install oss2
```

It can be installed. It is also very convenient to use, initialize with the permission ID and key, terminal name, etc., and then directly upload a file object. A file prefix is ​​specifically set for the automatic upload script. However, the OSS structure that was originally used to make the picture bed is also quite messy.

After the upload is complete, use python's cross-platform clipboard package `pyperclipto` copy the address to the clipboard.

```python
import pyperclip

pyperclip.copy(address)
```

Then the infrastructure is complete.

## Remove the white background

The more troublesome thing is that the background is transparent, because it is ugly to put a white background image on a web page of another color. The background of the photos in the Evernote scan mode is basically very white. Before you select the color directly in the PS (the tolerance is not large), delete it. This step can still be implemented programmatically.

Python's image processing library: PIL (Python Image Library). Step: Read in the file, convert to RGBA mode, traverse the pixel, if it is white, set the alpha channel to 0 and save. Code:

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

## Configure Path

This is basically done. In order to allow the script to be accessed in any path, create a new path D:\scripts\ and add it to the system Path so that the script inside can be executed directly. The python file and bat script in the directory, call python script by executing bat. Ossup.bat:

```bat
python D:\scripts\oss.py %1 %2
```

You're done.

Finally the full python file:

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

Result:

![example](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/ossup-example.png)
