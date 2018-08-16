---
layout: single
title:  "Failed to configure Site Recovery Manager with Platform Services Controller"
date:   2018-08-15 20:44:51 -0500
categories: vsphere
---

While upgrading Site Recovery Manager to 6.5 I got "Error 25212. Failed to configure Site Recovery Manager with Platform Services Controller"

![Site Recovery Manager Error 25212](/assets/images/srm-psc-error.png)

The installation log is named VMSrmInst.log and should be in the current user's temp directory, such as "C:\Users\Administrator\AppData\Local\Temp\VMSrmInst.log".  Looking through this log I found some more detail on the error.

```
2018-05-21T16:07:37.188-04:00 info vmware-dr[05516] [Originator@6876 sub=InfraNodeConfig] Creating service with service id: 7212e15b-6bbc-401a-bdca-35c105ce5c90
2018-05-21T16:07:37.251-04:00 error vmware-dr[05516] [Originator@6876 sub=InfraNodeConfig] Service with service id '7212e15b-6bbc-401a-bdca-35c105ce5c90' already exists in LookupService.  Failed to create service
.....
Error [25212]: Failed to configure DR server with the Infrastructure Node services.  Reason: lookup.fault.EntryExistsFault
```

The most common cause of this appears to be related to certificates, based on what I found in [VMware KB 2150750][1].  Following the resolutions in this KB article didn't help.  I also looked through [KB 2138411][2], but this didn't appear to be the problem.  No certificates had been replaced recently and SRM was connecting to the PSC, but it looked like it just wasn't registering the new version for some reason.  Looking through the Google search results it looks like there are a lot of certificate related issues when it comes to SRM connecting to the PSC.  Another article that is probably worth a look is [VMware KB 2109074][3].  Again I didn't think this was the problem in this particular case.

I opted to take a look at the registered services on the PSC and try unregistering the service.  [VMware KB 2043509][4] has some information on using "lstool.py" to view the registered services.  Note that it's suggested to contact [VMware Technical Support][5] to unregister any service.  Make sure you have a snapshot, don't do this in production, proceed with the appropriate level of caution, etc, etc...

To list the services registered on the PSC and then filter on the service ID shown in the error you can run lstool.py like this on the appliance:
```
root@psc01 [ ~ ]# /usr/lib/vmidentity/tools/scripts/lstool.py list --url 'http://localhost:7080/lookupservice/sdk' | grep '7212e15b-6bbc-401a-bdca-35c105ce5c90'  
```

To unregister the service:
```
root@psc01 [ ~ ]# /usr/lib/vmidentity/tools/scripts/lstool.py unregister --url 'http://localhost:7080/lookupservice/sdk' --user "administrator@vsphere.local" --password 'xxxxxxx' --id '7212e15b-6bbc-401a-bdca-35c105ce5c90' --no-check-cert
```

After unregistering the service and re-running the installation Site Recovery Manager was registered with the PSC and the upgrade completed successfully.

[1]: https://kb.vmware.com/kb/2150750
[2]: https://kb.vmware.com/kb/2138411
[3]: https://kb.vmware.com/kb/2109074
[4]: https://kb.vmware.com/kb/2043509
[5]: https://kb.vmware.com/kb/2006985
