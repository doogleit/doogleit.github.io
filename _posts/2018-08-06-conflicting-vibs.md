---
layout: single
title:  "The upgrade contains the following set of conflicting VIBs"
date:   2018-08-06 22:05:21 -0500
categories: vsphere
---

There are a lot of articles written on this subject that explain how to use esxcli to remove the conflicting VIBs. While that is usually the easiest way to work around this, I had a slightly different experience recently when I started upgrading a group of hosts to ESXi 6.5.  I downloaded the VMware ESXi 6.5 ISO from VMware's site and imported the image into Update Manager.  After runnning a scan I had a number of conflicting VIBs and oddly the event error in vCenter/Update Manager had the same one listed twice:

```
The upgrade contains the following set of conflicting VIBs:
LSI_bootbank_scsi-mpt3sas_04.00.00.00.1vmw-1OEM.500.0.0.472560
LSI_bootbank_scsi-mpt3sas_04.00.00.00.1vmw-1OEM.500.0.0.472560
VMW_bootbank_scsi-bnx2fc_1.78.78.v60.8-1vmw.650.0.0.4564106
```

While I was refilling my coffee and thinking about manually removing those VIBs it occurred to me that these were Dell hosts and had likely been originally installed using the Dell custom ISO.  I will typically use the OEM provided ISO to install ESXi when one is available, but had not really thought about using it for this upgrade.  I checked with Google for the right esxcli command to get this information and ran it on one of the hosts.

```
[root@esx01:~] esxcli software profile get
(Updated) Dell-ESXi-6.0U2-3620759-A02
   Name: (Updated) Dell-ESXi-6.0U2-3620759-A02
   Vendor: Dell
   Creation Time: 2018-07-26T00:28:30
   Modification Time: 2018-07-26T00:28:30
   Stateless Ready: False
```

Sure enough it was installed using a Dell image.  Here's another example of a newer image that now uses "DELLEMC": 

```
[root@esx02:~] esxcli software profile get
(Updated) DELLEMC-ESXi-6.0U3-7967664-A10
   Name: (Updated) DELLEMC-ESXi-6.0U3-7967664-A10
   Vendor: Dell
   Creation Time: 2018-06-28T19:01:59
   Modification Time: 2018-06-28T19:01:59
   Stateless Ready: False
```

Finally, here's sample output from a stock VMware image:

```
[root@esx03:~] esxcli software profile get
(Updated) ESXi-6.5.0-20180502001-standard
   Name: (Updated) ESXi-6.5.0-20180502001-standard
   Vendor: VMware, Inc.
   Creation Time: 2018-06-24T08:17:38
   Modification Time: 2018-06-24T08:17:38
   Stateless Ready: False
```

I've truncated the output above to just show the summary information.  The full output includes a lot of additional information including all of the VIBs installed and which VIBs have been updated.

I figured it was a good idea to upgrade the host using the Dell image profile, even though I didn't think it would make much of a difference.  Surprisingly, after I imported the Dell custom ISO into Update Manager, updated the baseline, and re-scanned the hosts there were no conflicting VIBs!  I haven't been able to reproduce this so I don't think it is a common problem, but if you run into this on some older hosts it might be worthwhile to check the image profile that is in use.

