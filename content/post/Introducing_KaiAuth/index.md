---
title: "Introducing KaiAuth"
date: 2020-03-20T23:10:00+08:00
lastmod: 2020-03-20T23:10:00+08:00
draft: false
tags: ["KaiOS", "KaiAuth"]
categories: ["KaiOS"]

---

> KaiAuth is a Google Authenticator alternative that works on KaiOS

![KaiAuth logo](logo.png)

I am not a native speaker of English, so my English is very bad. I hope you can forgive me for bad grammar or spelling mistakes.

# Install
## WebIDE
1. Download https://github.com/zjyl1994/KaiAuth/archive/master.zip .
1. Extract this zip to a folder.
1. Click "Open Packaged App" and select the folder from the previous step.
1. Click "Install and Run" button.
## OmniSD
1. Open https://github.com/zjyl1994/KaiAuth/releases/latest and download **kaiauth.zip**.
1. Copy this zip file to the apps directory of the memory card.
1. Open **OmniSD** and install this zip.

When you do the right thing by following the steps above,KaiAuth will appear in your phone menu.

# Use
## Basic functions
![Icon in menu](logo_in_menu.png)
### Main interface
![Main interface](main_interface.png)

Although these screenshots are in Chinese, don't worry, English versions are now available.

You can see a lot of TOTP codes on the screen, you can use the navigation keys to scroll them up, down, left and right.

There are two soft keys at the bottom, the left one means to add a new code, and the right one means to delete the currently selected code.

Before deleting, you will be asked if you want to delete, so you do n’t need to worry about mistakes.

Press the soft key named Add on the left, the page will switch to the scan QR code page.
### Scan QR code interface
![QR code scan](scan_qrcode.png)

Scanning the QR code will pop up an application for using camera permissions. Click Allow to enter the above interface.

Place the QR code in the camera area. If an available QR code is identified, it will automatically jump back to the main interface and add it to the code list.

## Advanced Features
Taking into account the actual needs, I have specially made some advanced features, which can be used in the form of short codes.

You need to enter the following short codes using the keyboard in the KaiAuth app. Do not use these instructions in the phone dialer.

### *#7370#
This is a reset shortcode for very old Nokia phones. The meaning here is to clear the data storage in the app. It will be very useful when you mess up your code list and need to re-import the correct data.
### *#0000#
This is also the short code that very old Nokia will use, and now its role is to query the KaiAuth version you are running.
### *#467678#
Load a file named kaiuth.json from the SD card into the code list, which is useful when you transfer an existing code list from another device, or use a previously prepared backup.
### *#397678#
Exporting the current code list to a file named kaiuth.json in the SD card is very useful when backing up or transferring data to a new device.
**Very important tip:** The exported data file contains sensitive information, please keep it in a safe place. Anyone who gets the data file can use these codes to access your account.

# Why i made it
I bought a Nokia 2720 Flip, which runs KaiOS. I have many accounts protected with Google Authenticator. I hope that I can still query these codes and log in to my account on my computer when I am away from my smartphone, so I go to the Internet to search for related tools.

Unfortunately, I didn't find related tools, some of them are only rough implementation of web pages, which need to use cursor to operate, which is very uncomfortable. Some don’t even have a basic feature to scan QR codes. It ’s very difficult to enter the two-step verification key accurately on the T9 keyboard of old phones. So I developed this KaiAuth myself, hoping that it can help someone who needs it.

If you like my work, please give me star on GitHub. https://github.com/zjyl1994/KaiAuth

Thanks for the help of Google Translate, I hope you can understand my bad English.