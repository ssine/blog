---
title: "个人自动化系统与信息看板"
date: 2021-04-17T00:51:21+08:00
---

有段时间比较关注数字健康和自动化，就折腾了一波相关的东西，搞了数据存储和看板。 回顾一下整个流程感觉像是个微型数据仓库，需要自己去获取数据源，设计存储格式并转移到数据库，然后把数据库里的信息展示出来。

# 自动化与数据收集

首先聊聊我关心的一些信息以及数据收集的过程。

## 安卓自动化工具 Tasker

虽然忘记了一开始为什么研究 Tasker，但是回过头来看 Tasker 是整个系统至关重要的一环，所以优先介绍。

首先是各类自动化工具的选择，不像 IOS 上有几乎一统天下的 ShortCuts 捷径 应用 ( 主要也是 IOS 权限管的严，官方给资源就是太子了 ) ，安卓上面的选择有很多，比较活跃的有 Tasker 和 Macrodroid, Tasker 是很久以前就存在的老牌应用了，社区相关的资源很多，不过 Macrodroid 的 UI 要比 Tasker 强不少。 因为重点还是要做事情，以及看重 Tasker 比较偏程序员的能力 ( js/shell 脚本、HTTP 相关 ) ，还是购买了 Tasker。

软件第一次打开的时候会有教程，下面简单描述一些比较重要的概念：

-   变量： 可以被访问和修改的全局变量，用于存储状态
-   任务： 相当于程序，由一组指令构成，指令可以做各种事情，比如请求 HTTP API
-   触发器： 用来判断何时执行任务的条件，可以分为：
  -   事件： 某件事情发生的时候，比如打开手机屏幕，扫描 NFC Tag
  -   位置： 进入某个位置时触发，需要开启定位，GPS->WLAN->基站，功耗和准确度都依次降低
  -   应用程序： 打开某个应用时触发
  -   时间： 在某个时间点触发
  -   状态： 进入或离开某个状态时触发，比如进入电量>10% 状态时
-   配置文件： 由触发条件和任务组成，在触发条件满足时执行任务的配置
-   项目： 顶级命名空间，可以管理多组上述变量、任务、触发器、配置文件

其他的信息结合后面的具体应用阐述。

## 日常打点

第一件想做的事情是记录 「每天各种事件发生的时间」 ，比如每天几点起床几点睡觉，什么时候上下班，等等。 醒来和入睡的时间要靠手环了，先说说通勤和工作的记录。

每天的日程可以概括成

```text
     通勤时间        工作时间           通勤时间
离家 <------> 到公司 <------> 离开公司 <------> 到家
```

只要记录这四个时间点，相关的时间就能得出了。 那么手机要如何得知这几个事件呢？

1.  通过位置信息，这是最直接的方式，但是高精度的 GPS 定位耗电很多，低精度的基站定位有不够准确，出门好久才反应过来。
2.  通过 Wifi 的连接与断开。 连接上家里的 Wifi=到家了，与家里的 Wifi 断开=离开家，公司也是一样。 不过这有个缺陷，比如在家的时候因故断开 Wifi 一阵子，就会被记为离开家又回来。
3.  Wifi 连接与断开+Wifi 变化。 这是 2 的增强版，如果我上一次断开的 Wifi 是家里的，这一次连接的 Wifi 是公司的，那么就可以确定这次连接 Wifi 等于来到公司了。 不过这仍然有个缺点，就是不好判定 Wifi 断开的时候是否真的是离开了 Wifi 所在地，因为你还不知道下一次连接的 Wifi 是什么 ( 只能等到下一次连接 Wifi 的时候再记录 ) 。

我目前的方式是利用第三种来判断进入地点的事件，并用扫 NFC Tag 的方式来判断离开地点，像是下班打卡。 其实要改成延迟记录的话纯 Wifi 就可以了，不过比较懒。

首先要维护「上一个 Wifi」的信息：

```text
Profile: Anon (23)
  Restore: no
  State: Not Wifi Connected [SSID:* MAC:* IP:* Active: Any ]
Enter: Anon (27)

Profile: Anon (25)
  Restore: no
  State: Wifi Connected [SSID:* MAC:* IP:* Active: Any ]
Enter: Anon (29)

Task: Anon (27)
  A1: Variable Set [Name:%LastWIFI To:%CurrentWIFI ]
  A2: Variable Set [Name:%CurrentWIFI To:. ]

Task: Anon (29)
  A1: Test Net [Type: Wifi SSID Data: Store Result In:%CurrentWIFI ]
```

就是在断开 Wifi 的时候更新 LastWifi，连接 Wifi 时给 CurrentWifi 赋值。

然后是回家和上班的记录：

```text
Profile: 记录：到家 (8)
  Cooldown: 600 Restore: no
  Variables: [ %action: has value ]
  State: Wifi Connected [SSID: 爷斋小二 MAC:* IP:* Active: Any ]
Enter: Anon (30)

Task: Anon (30)
  A1: Wait [MS:0 Seconds:5 Minutes:0 Hours:0 Days:0 ]
  A2: If [ %CurrentWIFI neq %LastWIFI ]
  A3: HTTP Request [Method: GET URL: https://tracker.example.com/life-tracer/log-action?action=%action]
  A4: End If

Profile: 记录：到公司 (14)
  Cooldown: 600 Restore: no
  Variables: [ %action: has value ]
  State: Wifi Connected [SSID: ByteDance Inc MAC:* IP:* Active: Any ]
Enter: Anon (28)

Task: Anon (28)
  A1: Wait [MS:0 Seconds:5 Minutes:0 Hours:0 Days:0 ]
  A2: If [ %CurrentWIFI neq %LastWIFI ]
  A3: HTTP Request [Method: GET URL: https://tracker.example.com/life-tracer/log-action?action=%action]
  A4: End If
```

在 Wifi 连接时如果发生变化就去请求搭建好的 API 记录时间，API 的事情后面会说。

至于离开家和离开公司，就靠 NFC 标签触发了：

```text
Profile: 记录：离公司 (16)
  Cooldown: 600 Restore: no
  Variables: [ %action: has value ]
  Event: NFC Tag [ID:048AFF32100289 Content: 下班 ]
Enter: Anon (9)

Task: Anon (9)
  A1: HTTP Request [Method: GET URL: https://tracker.example.com/life-tracer/log-action?action=离公司]
```

这需要 NFC 标签和支持 NFC 的手机 ( 现在基本都支持 ) ，向标签中写入特定的值，或是直接使用 ID 来确定标签，并在标签被扫时触发相应的事件。 NFC 标签使用的是以前 Mock Switch Amiibo 的时候用的 NTAG215 钱币卡， [淘宝链接](https://m.tb.cn/h.4ppvxjN) 。 安卓应用 NFC Tools Pro 是个很好用的 NFC 读取和写入的工具。

虽然 NFC 标签有时候显得比较鸡肋，只是打卡记录时间，点手机屏幕也可以，但是还是比按手机上的按钮更有仪式感，能够放在醒目的位置提醒你去扫一下，效率也比解锁手机打开 App 记录要高。

## 身体指标

有一些跟健康相关的指标，比如心率、睡眠时间和每日步数，适合用智能手环或者手表来收集。 不过要自动化的收集这些数据并存储起来挺麻烦的，首先很多设备本身就不支持数据导出，国外有很多支持 Google Fit 的设备，但是 Google Fit 只提供了一次性或者每两个月的导出，并不适合做看板。 华为的手表功能很多，各种数据算法也很准，可惜过于封闭，并不能拿到数据。 找了一圈发现小米手环有一个第三方应用 [Notify & Fitness for Mi Band](https://play.google.com/store/apps/details?id=com.mc.miband1) 很开放，可以导出数据，还支持通过 intent 与 Tasker 集成，就决定用小米手环和这个应用来做了。

首先要用正常的流程用小米手环给官方 app 小米健康配对，之后 Notify 应用可以与小米健康一起运行，或是取出小米健康 APP 的密钥，Notify 独立运行，Notify 应用有详细的指示。 因为小米健康很臃肿，就只用 Notify 了。

Notify 应用主要妙在可以通过安卓 intent 机制结合 Tasker 实现自动化。 Intent 是在不同的应用之间传递消息的机制。 支持的功能可以在官方文档 [Tasker integration](http://www.mibandnotify.com/help/tasker_send_intent.php) 处查看，主要有收发两类，一方面可以在各类事件发生时发出 intent，一方面可以在收到 intent 时做一些事情。 导出数据就用到了 「Export steps/sleep/heart/weight spreadsheet data」 功能，在接收到这个 intent 时会把步数、睡眠、心率、体重等信息导出成 csv 文件。

体重和体脂是买的小米体脂秤，也可以和 Notify 应用无缝结合，因此尽管体脂并不准，还是图方便用它了。

Tasker：

```text
Profile: Sync Miband (19)
  Restore: no
  Time: Every 3h
Enter: 导出手环数据 (6)

Task:     导出手环数据 (6)
  A1: Send Intent [Action: com.mc.miband.syncData Target: Broadcast Receiver ]
  A2: Wait [Seconds:30]
  A3: Variable Set [Name:%start_time To:%TIMEMS-24*2*60*60*1000 ]
  A4: Send Intent [Action: com.mc.miband.taskerExportAllSpreadsheetData Extra: start:%start_time Target: Broadcast Receiver ]
  A5: Wait [Seconds:30]
  A6: Run Shell [Command: curl -u user:password -T /storage/emulated/0/miband/mibandnotify/export/exportHeart.csv https://drive.example.com/remote.php/dav/files/sine/Backup/Fitness/exportHeart.csv ]
  A7: Run Shell [Command: curl -u user:password -T /storage/emulated/0/miband/mibandnotify/export/exportSleep.csv https://drive.example.com/remote.php/dav/files/sine/Backup/Fitness/exportSleep.csv ]
  A8: Run Shell [Command: curl -u user:password -T /storage/emulated/0/miband/mibandnotify/export/exportSleepDetails.csv https://drive.example.com/remote.php/dav/files/sine/Backup/Fitness/exportSleepDetails.csv ]
  A9: Run Shell [Command: curl -u user:password -T /storage/emulated/0/miband/mibandnotify/export/exportWeight.csv https://drive.example.com/remote.php/dav/files/sine/Backup/Fitness/exportWeight.csv ]
  A10: Run Shell [Command: curl -u user:password -T /storage/emulated/0/miband/mibandnotify/export/exportSteps.csv https://drive.example.com/remote.php/dav/files/sine/Backup/Fitness/exportSteps.csv ]
  A11: HTTP Request [Method: GET URL: https://tracker.example.com/sync/mongo?items=miband ]
```

上面的 Profile 每三小时执行一次，先发 intent 给 Notify 应用，让它从手环同步数据，等待 30 秒后发命令导出过去两天的数据，再等待 30 秒数据导出 ( 足够了 ) ，然后使用 Shell 命令 curl 将导出的 csv 文件发送到 Nextcloud 网盘（通过 Webdav 接口），最后请求服务器去解析并存储网盘上导出的数据。 

下面是解析导出的文件并存储到 MongoDB 的脚本，其中睡眠数据的解析比较麻烦，有多个睡眠阶段，是个零乱的数组，也不知道准不准：

```python
def add_heart_data():
    txt = get_fs_file_content('/Backup/Fitness/exportHeart.csv')
    df = pd.read_csv(StringIO(txt), sep=';')

    inserts = []
    for _, row in tqdm.tqdm(df.iterrows(), total=df.shape[0]):
        inserts.append({
            'ts': datetime.fromtimestamp(row['时间戳'] // 1000, timezone.utc).replace(microsecond=row['时间戳'] % 1000),
            'value': row['心率']
        })

    insert_docs('sine', 'heart_rate', inserts)

def add_step_data():
    txt = get_fs_file_content('/Backup/Fitness/exportSteps.csv')
    df = pd.read_csv(StringIO(txt), sep=';')

    inserts = []
    for _, row in tqdm.tqdm(df.iterrows(), total=df.shape[0]):
        inserts.append({
            'ts': datetime.fromtimestamp(row['时间戳'] // 1000, timezone.utc).replace(microsecond=row['时间戳'] % 1000),
            'value': row['步数']
        })

    insert_docs('sine', 'step', inserts)

def add_weight_data():
    txt = get_fs_file_content('/Backup/Fitness/exportWeight.csv')
    df = pd.read_csv(StringIO(txt), sep=';')

    inserts = []
    for _, row in tqdm.tqdm(df.iterrows(), total=df.shape[0]):
        inserts.append({
            'ts': datetime.fromtimestamp(row['时间戳'] // 1000, timezone.utc).replace(microsecond=row['时间戳'] % 1000),
            'value': row['体重']
        })

    insert_docs('sine', 'weight', inserts)

    inserts = []
    for _, row in tqdm.tqdm(df.iterrows(), total=df.shape[0]):
        inserts.append({
            'ts': datetime.fromtimestamp(row['时间戳'] // 1000, timezone.utc).replace(microsecond=row['时间戳'] % 1000),
            'value': row['Fat']
        })

    insert_docs('sine', 'fat', inserts)

def add_sleep_data():
    txt = get_fs_file_content('/Backup/Fitness/exportSleepDetails.csv')
    df = pd.read_csv(StringIO(txt), sep=';')

    events = []
    stage_map = {
        'wake': 5,
        'rem': 4,
        'stage1': 3,
        'stage2': 2,
        'stage3': 1,
        'stage4': 0,
    }
    miband_map = {
        '浅睡': 'stage1',
        '深睡': 'stage3',
        '清醒': 'wake',
        'REM': 'rem',
    }

    for _, row in df.iterrows():
        events.append((row['开始睡眠'], 'in', miband_map[row['睡眠类型']]))
        events.append((row['醒来'], 'out', miband_map[row['睡眠类型']]))

    events.sort(key=lambda e: e[0])

    state = 'wake'
    stage_cnt = {
        'wake': 1,
        'rem': 0,
        'stage1': 0,
        'stage2': 0,
        'stage3': 0,
        'stage4': 0,
    }
    inserts = []
    for idx, evt in enumerate(events):
        if evt[1] == 'in':
            stage_cnt[evt[2]] += 1
        else:
            stage_cnt[evt[2]] -= 1

        if idx == len(events) - 1 or events[idx + 1][0] > evt[0]:
            for k in reversed(stage_cnt.keys()):
                if stage_cnt[k] > 0 and k != state:
                    ts = datetime.fromtimestamp(
                        evt[0] // 1000, timezone.utc).replace(microsecond=evt[0] % 1000)
                    inserts.append({
                        'ts': ts,
                        'value': stage_map[k],
                    })
                    state = k
                    break

    insert_docs('sine', 'sleep', inserts)
```

Notify 还有在睡着和醒来时发送 intent 的功能，因此可以记录睡眠的时间，不过通常有十分钟左右的延迟：

```text
Profile: 记录：睡着 (3)
  Restore: no
  Variables: [ %action:has value ]
  Event: Intent Received [ Action:com.mc.miband.tasker.fellAsleep ]
Enter: Anon (31)
    
Task:     Anon (31)
  A1: HTTP Request [  Method:GET URL:https://tracker.example.com/life-tracer/log-action?action=%action ] 
  A2: Wait [ MS:0 Seconds:0 Minutes:15 Hours:0 Days:0 ] 
  A3: Media Control [ Cmd:Pause Simulate Media Button:On Package/App Name: 网易云音乐 Use Notification If Available:Off ] 

Profile: 记录：醒来 (4)
  Restore: no
  Variables: [ %action:has value ]
  Event: Intent Received [ Action:com.mc.miband.tasker.wokeUp ]
Enter: Trace POST (2)

Task:     Trace POST (2)
  A1: Wait [ MS:0 Seconds:10 Minutes:0 Hours:0 Days:0 ] 
  A2: HTTP Request [  Method:GET URL:https://tracker.example.com/life-tracer/log-action?action=%action ] 
```

睡着和醒来都会去请求 API 记录事件，睡着的时候会额外关闭正在播放的音乐。

## 数字健康

Digital Wellbeing 主要提倡不要被电子设备支配，在浪费很多不必要的精力。 具体到用法上就是去统计每个应用使用的时间了。

### 手机

首先是手机。 Tasker 自带了从安卓系统获取应用使用时间的功能，但是不知为何这个系统功能记录的时间令人发指的不准，而且缺乏文档，不能明确是进程运行时间还是屏幕使用时间，后来忍不了就决定自己用 Tasker 记录了：

```text
Profile: Phone Usage - Change (38)
  Restore: no
  Event: App Changed [ Output Variables:* Package:* ]
Enter: Anon (39)

Task:     Anon (39)
  A1: If [ %CurrentAPP neq . ]
  A2: Variable Set [ Name:%usageTime To:%TIMES-%APPOpenTime ] 
  A3: JavaScriptlet [ Code:let l = []
    try {
      l = JSON.parse(global("APPUsageList"))
    } catch (err) {
      l = []
    }
    l.push({ts: global("APPOpenTime"), name: global("CurrentAPP"), package: global("CurrentPackage"), time: global("usageTime")})
    setGlobal("APPUsageList", JSON.stringify(l)) Libraries: Auto Exit:On Timeout (Seconds):45 ] 
  A5: End If 
  A6: Variable Set [ Name:%CurrentAPP To:%app_name ] 
  A7: Variable Set [ Name:%CurrentPackage To:%app_package ] 
  A8: Variable Set [ Name:%APPOpenTime To:%TIMES ] 
  

Profile: Phone Usage - Screen (40)
  Restore: no
  Event: Display Off
Enter: Anon (41)

Task:     Anon (41)
  A1: If [ %CurrentAPP neq . ]
  A2: Variable Set [ Name:%usageTime To:%TIMES-%APPOpenTime ] 
  A3: JavaScriptlet [ Code:let l = []
    try {
      l = JSON.parse(global("APPUsageList"))
    } catch (err) {
      l = []
    }
    l.push({ts: global("APPOpenTime"), name: global("CurrentAPP"), package: global("CurrentPackage"), time: global("usageTime")})
    setGlobal("APPUsageList", JSON.stringify(l)) Libraries: Auto Exit:On Timeout (Seconds):45 ] 
  A4: End If 
  A5: Variable Set [ Name:%CurrentAPP To:. ] 
  A6: Variable Set [ Name:%APPOpenTime To:. ] 

Profile: Sync Phone Usage (43)
  Priority: 4 Restore: no
  Time:  Every 1h
Enter: Anon (44)

Task:     Anon (44)
  A1: HTTP Request [  Method:POST URL:https://tracker.example.com/phone_usage Body:%APPUsageList ] 
  A2: Variable Set [ Name:%APPUsageList To:[] ] 
```

具体内容看任务描述就可以了，通过几个全局变量记录当前打开的 APP 使用时间，并在切换 APP 的时候将上个 APP 的使用情况计入 APPUsageList，最后每隔一段时间将当前的 APPUsageList 发送到记录 API。 Tasker 的变量只支持字符串类型，因此要存储比较复杂的 JSON 格式就要写脚本处理了，好在 Tasker 支持 JavascriptLet 类型的任务，可以直接运行 JS 代码，这样就能实现比较复杂的逻辑。

### 电脑

Mac 上系统自带了时间统计（不过不能导出，但是问题不大，反正也能看到），Windows 上没有，有软件可以统计时间，但是也没有导出功能，只好自己写一个脚本统计了：

```python
import time
import win32gui
import win32process
import win32pdh
import uiautomation as auto
import psutil
import json
from urllib.parse import urlparse
from datetime import datetime, timezone
import requests
import copy
import logging
from win10toast import ToastNotifier
import os
import traceback
import re
from io import StringIO

logging.basicConfig(filename=os.path.join(
    __file__, '../logs.log'), level=logging.INFO, format='[%(asctime)s][%(levelname)s] %(message)s')

def get_chrome_active_domain(handle):
    ctl = auto.ControlFromHandle(handle)
    tmp = ctl.PaneControl(Depth=1, Name='Google Chrome').GetChildren()[
        1].GetChildren()[0]
    found = False
    for bar in tmp.GetChildren():
        last = bar.GetLastChildControl()
        if last and last.Name != '':
            found = True
            break
    if found:
        addr_bar = bar.GroupControl(Depth=1, Name='').EditControl()
        url = addr_bar.GetValuePattern().Value
        if '://' not in url:
            url = f'http://{url}'
        domain = urlparse(url).netloc
        if not re.match(r'([\S]+(\.[\S]+)+)|(\S+:\d+)', domain):
            return None
        return domain
    else:
        return None

process_name_map = {
    'chrome.exe': 'Chrome',
    'code.exe': 'VS Code',
    'explorer.exe': 'File Explorer',
    'powershell.exe': 'Powershell',
    'wechat.exe': 'Wechat',
    'element.exe': 'Element',
    'qq.exe': 'QQ',
    'cloudmusic.exe': '网易云音乐',
}

def check():
    wh = win32gui.GetForegroundWindow()
    title_text = win32gui.GetWindowText(wh)
    tid, pid = win32process.GetWindowThreadProcessId(wh)
    process = psutil.Process(pid)
    process.as_dict()
    process_name = str(process.name()).lower()
    child_path = []
    if process_name == 'chrome.exe' and ' - ' in title_text:
        domain = get_chrome_active_domain(wh)
        child_path.append(domain)
    elif process_name == 'code.exe':
        child_path.append(title_text.split(' - ')[-2])

    return {
        'process_name': process_name,
        'program_name': process_name_map.get(process_name, process_name),
        'window_title': title_text,
        'child_path': child_path,
        'ts': datetime.utcnow(),
    }

def state_changed(a, b):
    if a['process_name'] != b['process_name']:
        return True
    if len(a['child_path']) != len(b['child_path']):
        return True
    if any(map(lambda t: t[0] != t[1], zip(a['child_path'], b['child_path']))):
        return True
    return False

toaster = ToastNotifier()
toaster.show_toast("Desktop Usage Tracker", "Tracking started.", duration=10)

usages = []
last_app = None
last_send_time = datetime.now()
while True:
    try:
        tpassed = (datetime.now() - last_send_time).total_seconds()
        if tpassed > 60 * 10 and len(usages) > 0:
            try:
                logging.info(f'{tpassed:.2f} seconds passed, sending usages')
                res = requests.post(
                    'https://tracker.example.com/desktop_usage', json=usages)
                usages = []
                last_send_time = datetime.now()
                logging.info('sending done')
            except Exception as err:
                logging.error(f'failed to send usage: {err}')

        app = check()
        if not last_app or state_changed(last_app, app):
            tmp = copy.deepcopy(last_app) if last_app else copy.deepcopy(app)
            tmp['time'] = (datetime.utcnow() -
                                tmp['ts']).total_seconds()
            tmp['ts'] = tmp['ts'].replace(tzinfo=timezone.utc).timestamp()
            if tmp['process_name'] == 'chrome.exe' and len(tmp['child_path']) > 0:
                if not isinstance(tmp['child_path'][0], str) or not re.match(r'([\S]+(\.[\S]+)+)|(\S+:\d+)', tmp['child_path'][0]):
                    tmp['child_path'] = []
            if len(usages) > 0 and usages[-1]['ts'] == tmp['ts']:
                usages[-1] = tmp
            else:
                usages.append(tmp)
            last_app = app
    except Exception as err:
        logging.error(f'exception: {err}')
        s = StringIO()
        traceback.print_tb(err.__traceback__, file=s)
        logging.error(s.getvalue())
    time.sleep(0.5)
```

上面的脚本会每隔半秒钟获取当前活跃的窗口，如果窗口发生变化就记录上一个窗口的使用情况（开始，结束）。 对于比较常用的软件做了特殊处理，可以获取 Chrome 当前标签页的域名，以及 VSCode 正在工作的项目。 主要用了各种 Windows 相关的各种包。 最后会定期将使用情况上报给 API 进行数据记录。

想要让这个脚本开机启动的话，需要在 `C:\Users\<name>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup` 目录下新建一个 `vbs` 文件，内容是：

```vbs
DIM objShell
set objShell=wscript.createObject("wscript.shell")
iReturn=objShell.Run("""C:\ProgramData\Miniconda3\python.exe"" ""D:\Project\LifeTracker\DesktopTracker\track.py""", 0, TRUE)
```

这样每次启动都会在后台运行这个脚本了。

# 数据存储

前面主要描述了各种数据的来源，收集到数据之后都是通过 API 发送给服务器，服务器做一些简单的处理并存储进数据库。

数据库方面，大部分数据都是时间索引的，可以用关系型数据库、文档数据库或者时序数据库。 考虑到关系型数据库不够灵活，以及时序数据库功能比较特定而且对性能要求没那么高，就决定用比较方便使用的文档数据库 MongoDB 来做存储了。

MongoDB 的 docker-compose 文件：

```yaml
---
version: "2.1"
services:
  mongodb:
    image: mongo:latest
    container_name: mongodb
    environment:
      - PUID=1000
      - PGID=1000
      - MONGO_INITDB_ROOT_USERNAME=x
      - MONGO_INITDB_ROOT_PASSWORD=x
    volumes:
      - /home/sine/service/mongodb/data:/data/db
    ports:
      - 2012:27017
    restart: unless-stopped
```

API 服务用 Flask 搭建，主要功能就是接收 HTTP 传来的数据并写入：

```python
@app.route('/life-tracer/log-action')
def log_action():
    action = request.args.get('action')
    insert_docs('sine', 'daily_events', [{
        'ts': datetime.datetime.utcnow().replace(tzinfo=datetime.timezone.utc),
        'event': action
    }])
    send_qq_msg(123456, f'记录： {action}')
    return '200 OK'

@app.route('/sync/mongo')
def sync_mongo():
    items = request.args.get('items')
    items = items.split(',')
    for item in items:
        if item == 'miband':
            add_step_data()
            add_heart_data()
            add_sleep_data()
            add_weight_data()
        elif item == 'thermometer':
            add_thermometer_data()
        elif item == 'waka':
            add_wakatime_data()
    return '200 OK'

@app.route('/phone_usage', methods=['POST'])
def phone_usage():
    data = request.get_json(force=True)
    updates = []
    for d in data:
        try:
            ts = datetime.datetime.utcfromtimestamp(int(d['ts']))
            updates.append(UpdateOne({'ts': ts}, {'$set': {
                'ts': ts,
                'name': d['name'],
                'package': d['package'],
                'time': int(d['time']),
            }}, upsert=True))
        except Exception as err:
            print(err)
    bulk_write('sine', 'phone_usage', updates)

    return '200 OK'

@app.route('/desktop_usage', methods=['POST'])
def desktop_usage():
    data = request.get_json(force=True)
    updates = []
    for d in data:
        ts = datetime.datetime.utcfromtimestamp(d['ts'])
        updates.append(UpdateOne({'ts': ts}, {'$set': {
            **d,
            'ts': ts
        }}, upsert=True))
    bulk_write('sine', 'desktop_usage', updates)

    return '200 OK'

@app.route('/thermometer_report')
def thermometer_report():
    temperature = request.args.get('temperature')
    humidity = request.args.get('humidity')

    client['sine']['room_metrics'].insert_one({
        'ts': datetime.datetime.utcnow(),
        'temperature': float(temperature),
        'humidity': float(humidity),
    })

    return '200 OK'
```

`/life-tracer/log-action` 可以记录某个时间点发生的事件， `/sync/mongo` 用于解析上传到 Nextcloud 的数据并存储到数据库，这里还有处理 Wakatime 数据的脚本，可以统计每天写代码的情况， Wakatime 免费版 API 只提供七天的详细数据，这种方式可以长久保存。 `/phone_usage` 和 `/desktop_usage` 是数字健康一节保存相关脚本数据的 API，最后 `/thermometer_report` 用于存储在 [用树莓派做温湿度上报与红外控制](https://ssine.ink/posts/rpi-thermometer-and-ir/) 中提到的房间温湿度数据。

# 数据看板

前面完成了数据的 ETL 过程，还差最后一步：数据展示与分析。 这里就通过 Grafana 看板来实现了。

Grafana 的 docker-compose 文件：

```yaml
---
version: "2.1"
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - PUID=1000
      - PGID=1000
      - GF_PATHS_CONFIG=/var/lib/grafana/config.ini
    volumes:
      - /home/sine/service/grafana/data:/var/lib/grafana
    ports:
      - 2013:3000
    restart: unless-stopped
```

Grafana 的官方 MongoDB 数据源的插件需要付费的企业版才能使用，所以这里用社区的一个插件 [mongodb-grafana-backend](https://github.com/PhracturedBlue/mongodb-grafana-backend) ，不过这个插件没有签名，需要手动在 `config.ini` 配置文件中允许这个插件：

```ini
[plugins]
allow_loading_unsigned_plugins = grafana-mongodb-backend-datasource
```

虽然另一个 MongoDB 插件 [mongodb-grafana](https://github.com/JamesOsgood/mongodb-grafana) star 很多，但是它是通过令启一个 node 服务来做转发的，放在 Docker 里面极为不优雅，因此还是用前面那个不需要后端服务的插件了。 至于安装，参考 README 里面把对应的目录拷贝到 Grafana 的 plugin 下面，然后重启 Grafana 服务就可以了。

除此之外还用到了 Plotly panel 用来通过 js 画图，以及 Pie Chart 饼图插件，这两个都可以在 Grafana 的设置页搜索安装。

先放一张完整的图：

![img](dashboard.jpg)

下面看看其中几个面板的实现。

每日手机使用时长就是一个常规的查询

```json
[
  {"$match": {"ts": {"$gte": "$from", "$lt" : "$to"}}},
  {"$match": {"name": {"$ne": "一加桌面"}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m-%d", "date": "$ts"}},
    "time": {"$sum": "$time"}
  }},
  {"$project" : {"name" : "usage", "value" : "$time", "ts" : {"$toDate": "$_id"}, "_id" : 0}},
  {"$sort": {"ts" : 1}}
]
```

取出特定时间段的数据，过滤掉无意义应用，然后以天问单位求和，最后映射到插件的指定格式。

过去一段时间的手机应用使用时长的饼图：

```json
[
  {"$match": {"ts": {"$gte": "$from", "$lt": "$to"}}},
  {"$group": {
    "_id": "$name",
    "total": {"$sum": "$time"}
  }},
  {"$redact": {
    "$cond": [
      {"$gt": ["$total", 300]},
      "$$KEEP",
      "$$PRUNE"
    ]
  }},
  {"$sort": {"total": -1}},
  {"$match": {"_id": {"$nin": ["一加桌面", "Root Explorer", "CaptivePortalLogin"]}}},
  {"$project": {"name": "$_id", "value": "$total", "_id": 0}}
]
```

按时间筛选，按 name 求和，然后筛选出使用时间大于 5 分钟的应用，最后转换格式。

体重的折线图：

```json
[
  {"$match": {"ts": {"$gte": "$from", "$lt": "$to"}, "value": {"$gte": 50}}},
  {"$sort": {"ts": 1}},
  {"$project": {"name": "value", "value": "$value", "ts": "$ts", "_id": 0}}
]
```

很简单就不说了，需要注意的是如果返回的数据没有按照事件排序的话，鼠标 hover 的功能会比较错乱。

前面包含了柱状图、饼图、折线图的数据转换，都是自带插件可以比较方便实现的。 日常打点和事件时长就比较难做，因为 Grafana 还是关注于时序数据的展示，像图中的每天一条竖线的打点图和散点图就不是本行，需要靠外部插件 Plotly.js 来支持。

先看日常打点：

![img](daily_events.png)

每条竖线表示一天的 0 到 24 时，对应的事件用点在这条线上表示，这样可以展示很多天各种事件的走势。 实现上，数据源还是 MongoDB：

```json
[
    {"$match": {"ts": {"$gte": "$from", "$lt": "$to"}}},
    {"$project": {"name": "$event", "value": 1, "ts": "$ts", "_id": 0}}
]
```

在 Panel 设置中选择 Plotly Panel，有两个主要配置，一个 Layout 可以设置图表样式，一个 Script 可以自定义数据处理脚本。

Layout:

```json
{
  "font": {
    "color": "darkgrey"
  },
  "paper_bgcolor": "rgba(0,0,0,0)",
  "plot_bgcolor": "rgba(0,0,0,0)",
  "margin": {
    "t": 10,
    "b": 40
  },
  "xaxis": {
    "type": "date",
    "gridcolor": "rgb(46, 47, 49)"
  },
  "hovermode": "x",
  "yaxis": {
    "gridcolor": "rgb(46, 47, 49)"
  }
}
```

Script:

```js
let serieses = []

const getLocalISOString = (ts) => {
  d = new Date(ts)
  const tzOffset = d.getTimezoneOffset() * 60000
  return new Date(d - tzOffset).toISOString()
}

for (let s of data.series) {
  if (s.name) {
    let dates = s.fields[0].values.map(d => getLocalISOString(d).substr(0, 10))
    let values = s.fields[0].values.map(d => '2000-01-01 ' + getLocalISOString(d).substr(11, 8))
    serieses.push({
      name: s.name,
      x: dates,
      y: values,
      type: 'scatter',
      mode: 'markers',
      marker: { size: 12 },
    })
  }
}

return {
  data: serieses,
  layout: {
    xaxis: {
      tickformat: '',
    },
    yaxis: {
      tickformat: '%X',
      range: ['2000-01-01 23:59:59', '2000-01-01 00:00:00']
    }
  }
}
```

脚本做的事情主要是遍历所有的事件，把每个事件的 x 值设置为天， y 值设置为当天的时间。 有一个很让人难受的问题，就是鼠标悬停在对应列的时候，当天的同类事件只有第一个会显示 label，也就是说如果当天入睡多次，只有第一次的入睡时间会被显示出来。 这个问题在官方仓库有一个 issue 提到，不过没有人解决，但是我很难受，就自己简单改了一下，使得同一列可以显示多个同类型的 label，具体的 commit 在 [这里](https://github.com/ssine/plotly.js/commit/adda7fae1358443e5a36b1eafb91541b6c4b61df) ，重新编译，覆盖 Grafana 插件目录下的对应文件就可以使用了，同样在原 issue 下面 [反馈了一下](https://github.com/plotly/plotly.js/issues/4294#issuecomment-755398700) 。

下面是各种事件时长的统计分布，横轴时事件的开始时间，纵轴是事件的结束时间。 这样在 `y = - x + b` 线上的点就是事件耗时为 b 的 “等时线” ，多个点就能方便的看出各种时间的统计规律，比如工作：

![img](event_work.png)

和睡觉：

![img](event_sleep.png)

。 实现上数据源和布局跟上面差不多，数据处理的脚本：

```js
const getLocalISOString = (ts) => {
  d = new Date(ts)
  const tzOffset = d.getTimezoneOffset() * 60000
  return new Date(d - tzOffset).toISOString()
}

let series = []

for (let hour of [1, 3, 6, 9, 12]) {
  let duration = hour * 3600000
  let y_start = duration
  let x_end = (24 - hour) * 3600000
  series.push({
    name: `${hour} 小时`,
    x: ['2000-01-01 00:00:00', '2000-01-01 ' + (new Date(x_end).toISOString()).substr(11, 8)],
    y: ['2000-01-01 ' + (new Date(y_start).toISOString()).substr(11, 8), '2000-01-01 23:59:59'],
    opacity: 0.5,
    mode: 'lines',
    line: {
      width: 1,
    }
  })
}

const plotEvent = (name, entryNames, leaveNames) => {
  let events = []

  for (let s of data.series) {
    if (entryNames.includes(s.name) || leaveNames.includes(s.name)) {
      events = events.concat(s.fields[0].values.map(d => ({
        name: s.name,
        ts: d
      })))
    }
  }
  
  events.sort((a, b) => a.ts - b.ts)
  
  let lastTime = null;
  let xs = [], ys = [];
  
  for (let ev of events) {
    if (entryNames.includes(ev.name)) {
      lastTime = '2000-01-01 ' + getLocalISOString(ev.ts).substr(11, 8)
    } else if (lastTime) {
      xs.push(lastTime)
      ys.push('2000-01-01 ' + getLocalISOString(ev.ts).substr(11, 8))
      lastTime = null
    }
  }
  
  series.push({
    name: name,
    x: xs,
    y: ys,
    type: 'scatter',
    mode: 'markers',
  })
}

plotEvent('sleep', ['睡着'], ['醒来'])
plotEvent('work', ['到公司'], ['离公司', '到家'])

return {
  data: series,
  layout: {
    xaxis: {
      tickformat: '%X',
      title: {
        text: '开始时间'
      }
    },
    yaxis: {
      tickformat: '%X',
      autorange: 'reversed',
      title: {
        text: '结束时间'
      }
    },
    hovermode: 'closest'
  }
}
```

这里就比较麻烦，首先要根据一个状态的开始时间与结束时间找出所有的状态持续区间，然后每个区间一个点，给 x 和 y 算出开始时间和结束时间，最后还要把所有日期统一成同一天。

# 总结

做这些事情主要是出于兴趣（或许还有对生活的热爱？），回过头来看，也是一段很长的旅程，做了各种各样的事情并把它们依次整合起来。 在这个过程中也写了很多脚本，Tasker 尤其有意思，任务是把手机作为运行环境的。 这或许也是我写的项目里面与现实世界最相关的一次。 同时也对生活、精力、健康的管理做了很多的思考，希望这个系统可以伴随我一段时间吧。
