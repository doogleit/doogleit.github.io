---
layout: single
title:  "Storage DRS and RDMs"
categories: vsphere
tags: vsphere storage drs
---

To summarize, store RDM mapping files on a dedicated datastore outside of any SDRS cluster.  This is one of those things that I wanted to write about with the hope that I'll remember it in the future or at least be able to refer back to this article.  It's very rare anymore that I have to work with RDMs, but I'm currently managing an environment where a few of the VMs still require RDMs.  It's not often that I need to do maintenance on these and even rarer that a storage vMotion is needed.  While I was recently tackling a long overdue upgrade to VMFS6, I ran into this problem where I could not storage vMotion a VM with a mix of VMDKs and RDMs.

When trying to migrate the VM to a new volume I was getting a SDRS fault indicating that there was insufficient disk space.

![Migrate RDM](/assets/images/rdm_svmotion.png)

Clicking on the highlighted "View SDRS faults…" showed insufficient disk space on datastore for each of the large RDMs attached to the VM.

![SDRS Faults](/assets/images/rdm_svmotion_faults.png)

I knew there was enough disk space for the VMDKs and suspected that it was including the space used by the RDMs.  These are physical mode RDMs and everything that I was finding online indicated that the RDM mapping files should be migrated to the new datastore with no problem.  VMware has a comprehensive KB article on this subject: 

[Migrating virtual machines with Raw Device Mappings (RDMs) (1005241)][1]

As far as I could tell migrating the RDM pointers should not be an issue and because they are physical mode it isn't possible to convert them to VMDKs on the destination datastore.  Since the first screenshot indicates that this is an SDRS fault I decided to try moving the datastore out of the Datastore Cluster.  Sure enough, this allowed me to migrate the VM, RDM mappings and all, to the new datastore.

After confirming it was SDRS, I found this Vmware KB article that describes the problem and resolution.

[Storage DRS generates only one datastore as initial placement recommendation in clusters with physical RDM (2148523)][2]

It seems that my Google-Fu was not particularly strong on this day.  If I had thought to include Storage DRS in my earlier searches I probably would have found this article before narrowing down the problem and saved some time in troubleshooting.  Fortunately, it looks like this is no longer an issue in vSphere 6.7!

[1]: https://kb.vmware.com/kb/1005241
[2]: https://kb.vmware.com/kb/2148523