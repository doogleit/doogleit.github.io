---
layout: single
title:  "NFS Advanced Settings and vCheck"
date:   2017-12-28 17:09:02 -0500
categories: vsphere esxi nfs nas vcheck
---
While working on a new plugin for the popular [vCheck][vcheck-vsphere] tool, I wanted to find the most common recommendations for the ESXi advanced settings pertaining to NFS/NAS datastores.  I knew that these varied some based on the storage vendor, but thought there might be enough common ground to offer some sensible defaults in the plugin.  I found that some of the settings can also vary based on ESXi version and in the end I erred on the side of caution and submitted the plugin as disabled by default.  My thought process here was to insure that a vSphere Admin needs to actively configure it and it won't make sub-optimal recommendations for some environments and not others.  This post is mostly to summarize what I found and list what I thought were some of the more useful references.

I created a table to easily compare the settings based on ESXi version and vendor.

Setting | ESXi 6.0u3 Defaults | ESXi 6.5 Defaults | VMware KB 2239 | VMware Whitepaper | NetApp | Tintri | Oracle |
------------------------|-----|-----|------|------|-----|------------|-----|
Net.TcpIpHeapSize	      | 0	  | 0	  | 32	 |	-   | 32	| See KB 2239| 32	 |
Net.TcpIpHeapMax 	      | 512	| 512	| 1536 |	-   | 512	| See KB 2239| 512 |
NFS.MaxVolumes	        | 8	  | 8	  | 256	 |	-   | 256	| See KB 2239| 256 |
NFS.HeartbeatMaxFailures| 10	| 10	| -    | 10	  | 10	| 10  	     | 10	 |
NFS.HeartbeatFrequency	| 12	| 12	| -    | 12	  | 12	| 12	       | 20  |
NFS.HeartbeatDelta	    | 5	  | 5		|	-	   | -    | -   | -  	       | 12  |
NFS.HeartbeatTimeout	  | 5	  | 5		| -    | 5	  | 5   | 5          | 5   |
NFS.SendBufferSize	    | 264	| 264	| -    | -    | -   | -          | 264 |
NFS.ReceiveBufferSize	  | 256	| 256	|	-	   | -    | -   | -          | 256 |

To keep the table simple I only included the settings for vSphere/ESXi 6.0 and up.  ESXi 5.5 and lower have lower recommendations for some of the settings.  Consult the references linked below for details.

References:
* KB 2239: [Increasing the default value that defines the maximum number of NFS mounts on an ESXi/ESX host (2239)][vmware-kb]
* VMware: [Best Practices for Running VMware vSphere® on Network-Attached Storage (NAS)][vmware-wp]
* NetApp: [VMware vSphere with ONTAP (NetApp)][netapp-bp]
* Tintri: [Tintri VMstore with VMware Best Practices Guide][tintri-bp]
* Oracle: [Best Practices for Configuring Oracle ZFS Storage Appliance and VMware vSphere 6.x with NFS Protocol][oracle-bp]

A couple of other good articles - this one is now somewhat dated (2009), but still very useful:
[A “Multivendor Post” to help our mutual NFS customers using VMware][multivendor-post]

A newer post (2015) including information on vSphere 6 and NFS 4.1 support:
[Best Practices running VMware with NFS][vmguru-bp]

One thing I think everyone can agree on is that it's important to have all of your hosts configured the same.  If you're interested in using [vCheck][vcheck-vsphere] to report on this download the latest version and enable the "Host NFS Settings" plugin.

[vmware-kb]: https://kb.vmware.com/s/article/2239
[vmware-wp]: https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/vmware-nfs-best-practices-white-paper-en-new.pdf
[netapp-bp]: https://www.netapp.com/us/media/tr-4597.pdf
[tintri-bp]: https://www.tintri.com/sites/default/files/field/pdf/whitepapers/tintri-vmstore-with-vmware-best-practices-guide-white-paper.pdf
[oracle-bp]: https://community.oracle.com/servlet/JiveServlet/downloadBody/1013472-102-1-155091/VMware_vSphere6.x_NFS.pdf
[multivendor-post]: http://virtualgeek.typepad.com/virtual_geek/2009/06/a-multivendor-post-to-help-our-mutual-nfs-customers-using-vmware.html
[vmguru-bp]: https://www.vmguru.com/2015/12/best-practices-running-vmware-with-nfs/
[vcheck-vsphere]: https://github.com/alanrenouf/vCheck-vSphere
