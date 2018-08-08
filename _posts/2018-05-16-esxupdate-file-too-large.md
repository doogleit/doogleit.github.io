---
layout: single
title:  "Updating ESXi 6.0 - [Errno 27] File too large "
date:   2018-07-30 17:24:39 -0500
categories: vsphere
---

I've been upgrading a lot of ESXi 6.0 hosts to 6.5 lately.  As much as I'd like to go to vSphere 6.7, a lot of the client plugins and 3rd party solutions just don't support it yet. One of the hosts was not updating via Update Manager and the events being logged in vCenter weren't very helpful.  After digging through esxupdate.log on the host, this appeared to be the most likely culprit:
```
esxupdate: 42890: HostImage: ERROR: Failed to send vob install.stage.error: [Errno 27] File too large
esxupdate: 42890: root: ERROR: InstallationError: (<...insert long list of VIBs here...>, 'Cannot locate source for payload a of VIB VMware_bootbank_esx-tboot_6.0.0-3.57.5050593 ')
DEBUG: Live image has been updated but /altbootbank image has not. This means a reboot is not safe.
```
I went through the steps in several VMware KB articles, such as [Investigating disk space on an ESX or ESXi host (1003564)][1] and [Creating a persistent scratch location for ESXi 4.x/5.x/6.x (1033696)][2].  The problem did not really appear to be related to disk space and there were some other less relevant search results that I looked through, but eventually I stumbled upon this blog article about an [ESXi 6.0 fix for corrupted host imageprofile][3].  He does a great job of explaining the problem and resolution in detail so I'm just going to briefly summarize here, mostly for my own reference.

If you check the size of the bootbank images on the host, they should be roughly the same size and 50+ KB in size.  This is from a normal 6.0 host:
```
[root@localhost:~] ls -l /bootbank/imgdb.tgz
-rwx------    1 root     root         51306 May  1  2017 /bootbank/imgdb.tgz
[root@localhost:~] ls -l /altbootbank/imgdb.tgz
-rwx------    1 root     root         50635 May  1  2017 /altbootbank/imgdb.tgz
```

The problem is evident when either of these is drastically smaller, roughly 100 bytes in this case.  The resolution is to copy imgdb.tgz from a known good host that is the same version.  I ended up having to do this on a few different hosts.

I've since found that there is a VMware KB article that describes a slightly different error message, but has a fairly simliar fix outlined in its resolution:  ["esxupdate error code 15" error after patching an ESXi host (2016147)][4].

[1]: https://kb.vmware.com/kb/1003564
[2]: https://kb.vmware.com/kb/1033696
[3]: https://www.provirtualzone.com/esxi-6-0-fix-corrupted-host-imageprofile/
[4]: https://kb.vmware.com/s/article/2016147
