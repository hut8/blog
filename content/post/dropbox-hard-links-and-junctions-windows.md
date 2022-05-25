+++
date = "2015-09-27T17:00:45-04:00"
draft = false
title = "Using Dropbox on Windows with Junctions and Hard Links"
tags = [ "dropbox", "file-systems", "windows", "source-control", "emacs" ]
icon = "windows.svg"
+++

[Dropbox](https://db.tt/NtB1w0kg) is a reliable and well-known service for synchronizing your files. But one shortcoming is that there is no way to selectively synchronize files outside of the main "Dropbox" folder. This matters when you are using Dropbox to synchronize files between computers in which other software expects those files to be in a certain place. In Windows, there is a very simple way to fix this.

<!--more-->

[GNU Emacs](https://www.gnu.org/software/emacs/) is a great example of a use case. Recently I've been forced to use Windows, which I now just see as a bootloader for emacs. I want to use Dropbox to backup and synchronize my `.emacs` file and my `.emacs.d` directory. These techniques are applicable if you're using a different system for synchronizing files, including source control such as `git`. If you just try to create a Windows shortcut (`.lnk` file), most software other than Windows Explorer won't follow the link.

## Junctions and Hard Links

NTFS has two functions that help us here: [junction points](https://en.wikipedia.org/wiki/NTFS_junction_point) and [hard links](https://en.wikipedia.org/wiki/Hard_link). From Windows Explorer, neither of these can be created, so you'll have to use a CLI utility to make them. In these examples I have the `.emacs` file and `.emacs.d` directory right in my home folder (`C:\Users\User`) and want to store them in my Dropbox while keeping the original files.

### Junctions

Junctions are symbolic links for directories. They only work on the same filesystem.

There is a utility shipped with Windows 8, Windows Server 2008, Windows Server 2012 and Windows Vista (but curiously absent in Windows 7) called
[mklink](https://technet.microsoft.com/en-us/library/cc753194.aspx) which streamlines creating junctions and hard links. But because I'm on Windows 7, I searched for this utility and it wasn't present on my hard drive. I do have all the latest Windows updates, so I can only assume that some sites I found that insist this exists on Windows 7 are wrong. Fortunately, [sysinternals](https://technet.microsoft.com/en-us/sysinternals/default) has a utility called [junction](https://technet.microsoft.com/en-us/sysinternals/bb896768.aspx) which exposes this functionality. So, to do what I want, it's just:

```shell
junction C:\Users\User\Dropbox\Windows\.emacs.d C:\Users\User\.emacs.d
```

Note the `destination` `source` order, which is the reverse of how real operating systems do it.

### Hard Links

It seems as though all recent versions of Windows have [fsutil.exe](https://technet.microsoft.com/en-us/library/cc753059.aspx), but if you have `mklink`, that can do this too. Strangely, `fsutil.exe` can query and delete reparse points (of which junctions are a type) but cannot make them.

```shell
fsutil hardlink create C:\Users\User\Dropbox\Windows\.emacs C:\Users\User\.emacs
```

### On the other machines

If you're doing this to synchronize your files, rather than just to back them up, then repeat the above commands, replacing the source and destination on those machines.
