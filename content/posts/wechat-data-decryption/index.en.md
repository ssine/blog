---
title: "WeChat History Extraction"
date: 2020-04-03T00:17:06+08:00
---

__Note: This article is mainly translated by Google translator, please turn to the Chinese version for more accurate expression if possible.__

Recently, I need to share some chat records with others. Because the large amount of screenshots is too troublesome, I want to take the WeChat data directly and back it up by the way. Follow the common process, take the EnMicroMsg.db file, find the password, and export. However, when the database password was obtained, the mobile phone imei used to register WeChat was untestable, and the current mobile phone failed. Therefore, we can only use brute force cracking. See below for details.

The whole process is divided into three steps:

1. Extract the chat message database
2. Get database password, decrypt
3. Export chat history

# Extract the chat message database

WeChat user messages are stored in sqlite3 files, located in`/data/data/com.tencent.mm/MicroMsg/{hash}/EnMicroMsg.db` . As long as it can be taken out, there are not many obstacles behind. However, owning`/data/data/com.tencent.mm` The only users with folder access permissions are: 1. Tencent development user 2. root. Since my phone has been rooted, it is easy to export this file. Here are a few ideas for non-root users:

## Back up to a system with root privileges

Since the machine does not have root permission, the chat history of the current mobile phone will be transferred to a system with root permission through WeChat chat history migration. The most reliable way is to find a rooted mobile phone. If you don't have such a device, you can also install an Android emulator on your computer. The emulator usually has root permissions. I have tried the Night God Simulator, but it only supports Android 4, which is already an ancient version, and it crashed after logging in to the account on WeChat. In addition to the night god, there are many simulators to try, and readers can find them by themselves.

## With the help of the manufacturer's backup function

Some mobile phone manufacturers provide system-level backup software functions. You can use this function to obtain a backup of the WeChat data storage directory, and then unpack the backup file to obtain this information. I know of Xiaomi, but I haven't actually tried it. Samsung mobile phones used to support this feature, but it was later cancelled, probably for safety reasons.

## Fake developer users

Since the developer can access the data directory of his application, he can also pretend to be a developer and copy the data inside through the adb shell. According to Android's security policy, only applications packaged in development mode can use adb shell to access the data directory. Therefore, it is necessary to obtain a WeChat installation package in the development state. I have tried to decompile WeChat, change the development mode mark in it, and package it back to install, but the process is extremely cumbersome, and because the signature is different during the second packaging, it cannot replace the previous WeChat application. Even if it is installed, the previous WeChat application cannot be obtained. Data. But I have no experience in Android development, maybe an experienced developer can do it.

Finally, if you don't have root before but you want to be rooted in order to obtain chat records, you must carefully consider it, because there is a step in the rooting process that needs to delete all data, and the consequences of not deleting forcibly rooting are unpredictable.

# Get database password and decrypt

Taken out in the previous step`EnMicroMsg.db` The database file is encrypted, and the encryption method is SQLCipher V2. Here are two ways to obtain the password.

## Generate password via IMEI

Many blogs on the Internet have mentioned the method of generating the WeChat database password, which is the first 7 digits of the MD5 value after the mobile phone's IMEI number and WeChat uin are spliced together. MD5 is in lowercase letters.

The first is IMEI, dial on the phone`*#06#` You can check, each card slot has an IMEI (take the first 14 digits), and each mobile phone has an MEID. Try it out. However, one thing that hurts is that the IMEI may not be the current mobile phone. I also saw that it is the IMEI of the mobile phone used when registering WeChat.

The rest is uin, similar to the id of a WeChat user, located in`/data/data/com.tencent.mm/shared_prefs/auth_info_key_prefs.xml` ：

<img src="./auth_info_key_prefs.jpg">

Regardless of whether uin is a negative number or not, just concatenate it as a string and IMEI. Then take the string IMEI + uin to calculate the MD5 value (you will find an online tool for calculation in Baidu), and take out the first 7 digits (lowercase) to be the password.

## Brute force

The password I got through the process in the previous section was wrong, and I even tried it with the mobile phone I used in high school, but it also failed. However, the password has only 7 digits and the character set is also limited (MD5 is a hexadecimal string), so there are only $16^7$ possibilities in total, and the amount of calculation is acceptable for brute force cracking.

There are ready-made tools on the Internet, so I won’t make wheels:[EnMicroMsg.db-Password-Cracker](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker) , This program uses CPU, my i7-6700HQ runs about 200 lines/s, and it takes 15 days.

This length of time is still a bit unacceptable, but fortunately, there is an issue under this warehouse.[CUDA version completed but won't release](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker/issues/10) Here, there is a user who speaks honestly and said that he would not release his own GPU version, but he released it:[SQLCipher-Password-Cracker-OpenCL](https://github.com/whiteblackitty/SQLCipher-Password-Cracker-OpenCL) . To run it, you need OpenCL + CUDA environment. There are more detailed configuration steps in the project README. It took a lot of effort to set up the environment. Due to various problems with CUDA, it took a day to toss.

The speed of this program is quite impressive, my GPU is GTX 1060, the average speed is 190k/s, and it only takes 20 min in the worst case.

After I ran, I found that I didn’t find the password.[Dose not work on my database in Feb. 2018](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker/issues/6) The reason was found in the WeChat version 7.0, which changed the read-write head of the database, which caused a problem in determining the success of the decryption. This is in the CPU version of the program.[commit 20eb4](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker/commit/20eb4f9bd8bb440bcc6817b4b7981b8bf58478a4) It was fixed. However, the GPU version has not changed, so it needs to be corrected manually:

Will`Lib/pbkdf2-sha1_aes-256-cbc.cl` Line 752 of the file ([link](https://github.com/whiteblackitty/SQLCipher-Password-Cracker-OpenCL/blob/7dc8147c315d625426e79673a5dc5bad7f0177b3/Lib/pbkdf2-sha1_aes-256-cbc.cl#L752)) change into

```cpp
if(((uint)(data[5] ^ iv[5])==0x40) && ((uint)(data[6] ^ iv[6])==0x20) && ((uint)(data[7] ^ iv[7])==0x20))
```

. After running again, the password was successfully obtained.

## Database decryption

In the case of brute force cracking, the program will save a decrypted database when the password is obtained. If it is a calculation method, you can refer to[wechat-dump](https://github.com/ppwwyyxx/wechat-dump) The project provides a script for decrypting using IMEI and uin.

# Export chat history

The decrypted database is an ordinary sqlite3 file, which can be opened with many software. SQLiteStudio is recommended here.

<img src="./sqlitestudio.png">

There are all chat records in the message table. Of course, no matter what software is used, the readability is very poor. A program is needed to display them separately according to chat groups/persons.

[wechat-dump](https://github.com/ppwwyyxx/wechat-dump) The library is a good library for extracting messages from the database and categorizing and exporting them. It needs to be run in the linux + python 2 environment. Please refer to the project README for the specific configuration.

List all contacts:

```bash
./list-chats.py decrypted.db
```

Export all chat records as text files:

```bash
./count-message.sh output_dir
```

Render a conversation as an html file:

```bash
./dump-html.py "<contact_display_name>"
```

The exported html file can be opened with a browser, or it can be further printed as pdf.

<img src="./result.jpg">

---

The data generated by the user does not belong to the user, and it is ridiculous. The design of the application is not centered on the user experience, but centered on the user's wallet. Such strict restrictions on chat data can greatly increase the migration cost of WeChat users, because users cannot import chat records into other software. However, this is extremely unfriendly to users. The chat history roaming function requires payment, and the highest level of roaming cannot save all chat records, so users cannot back up data outside of the mobile phone at all, and there will be no data if there is a problem with the mobile phone. Up. I have always hated behaviors like QQ and WeChat, but the last mobile phone has been used for a long time. If you want to root, you will lose all your data. If you want to use the Android emulator, you will find problems frequently. Therefore, the first thing I do after changing the phone is to root, risking the security of the device, not for all kinds of plug-ins, just to retrieve my own data.
