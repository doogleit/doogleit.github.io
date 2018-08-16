---
layout: single
title:  "Fixing the Site Recovery Manager Site ID"
date:   2018-08-18 10:49:11 -0500
categories: vsphere
---

I was recently upgrading Site Recovery Manager to 6.5 and realized that I had entered the site name incorrectly during the upgrade process.  After the installation initially failed and the installer rolled back its changes, on subsequent attempts to run the installer I was prompted to enter all of the installation info, e.g. the PSC, vCenter, Local Site Name, Plug-in ID, etc. If you didn't do the original isntallation or it's been a while you might not know exactly what should be entered for all these options.  In this case I got the site ID from the SRM plugin in the vSphere Web Client under the Site, Summary, SRM ID. The format looks something like: com.vmware.vcDr-<my site name as entered in installer>

This site ID needs to match at both of your sites or on the paired SRM/vCenter.  I accidently entered "Dr-MySite" as the site name, which made the SRM ID "com.vmware.vcDr-Dr-MySite" instead of just "com.vmware.vcDr-MySite".

To fix this I edited the ID in two SRM configuration files, "extension.xml" and "dr-vmware.xml".  These two KB articles, [VMware KB 1037446][1] and [VMware KB 2057962][2], more or less pointed me in the right direction even though they were not exactly my problem.  I already had a snapshot from prior to starting the upgrade, but I took another one here before editing the files just in case.  I stopped the SRM service and opened Windows Explorer to "C:\Program Files\VMware\VMware vCenter Site Recovery Manager\config".

In both "extension.xml" and "dr-vmware.xml" there is a \<key\> value containing this ID.  I changed both to remove the extra "Dr-" that I had entered.

Before:
```
<extension>
	<key>com.vmware.vcDr-Dr-MySite</key>
....
</extension>
```

After:
```
<extension>
	<key>com.vmware.vcDr-MySite</key>
....
</extension>
```

After saving both files I started the Site Recovery Manager service back up and was able to verify that the correct SRM ID was shown on the summary tab for the site in the vSphere Web Client.  I had to Reconfigure Pairing under the Actions menu to fix pairing between the two sites and after that SRM was looking happy at both sites.

[1]: https://kb.vmware.com/kb/1037446
[2]: https://kb.vmware.com/kb/2057962

