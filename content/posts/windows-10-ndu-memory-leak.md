+++
date = "2015-11-22T11:21:56-05:00"
draft = false
title = "Windows 10 NDU Memory Leak"
tags = [ "dropbox", "file-systems", "windows", "source-control", "emacs" ]
icon = "windows.svg"
+++

## Symptom

In Windows 10 (with a fresh install), "System" uses up a huge amount of memory

## Solution

### Try to disable network data usage via `regedit`:

* `HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\Ndu`
* Change `Start` (DWORD) to `4`

![Regedit](/img/regedit-ndu.png)

### Disable SuperFetch:

![Services Console](/img/services.png)

![superfetch](/img/superfetch.png)

### Update networking drivers:

![Device Manager](/img/device-manager-network-adapters.png)

![Driver Update](/img/driver-update-ethernet.png)


## Cause

Here's how I found this memory leak

* Download [Windows Driver Kit](http://go.microsoft.com/fwlink/p/?LinkId=317353)
* Start `C:\Program Files (x86)\Windows Kits\10\Tools\x64\poolmon.exe` as an administrator
* Hit "b" to sort by bytes (descending)
* ![poolmon.exe screenshot](/img/windows-10-ndu-leak.png)
