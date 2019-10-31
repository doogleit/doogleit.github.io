---
title:  "vSphere Profile-Driven Storage Errors"
date:   2019-10-30 17:03:23 -0400
categories: vSphere
tags: vcenter
---

This is another strange problem that I experienced recently.  Restarting the Profile-Driven Storage service fixed all the errors, but diagnosing them was not so straight forward.  The first error message was pretty clear that it was related to "policy based management" or PBM.

![PBM Error](/assets/images/pbm-error.png)

A quick search revealed several KB articles with similar PBM type errors, all of them suggesting to start or restart the profile-driven storage service.

* [Cloning a virtual machine fails with the error: A general system error occurred: PBM error occurred during PreCloneCheckCallback (2118557)][1]
* [Error: "PBM error occurred during PreCreateCheckCallback: Connection refused" while creating protection group (53707)][2]

Restarting the service fixed the issue on the above vCenter.  On a different vCenter I later found that there were a number of migration errors.  Hosts were not able to enter maintenance mode due to DRS related errors.

![DRS Migration Error](/assets/images/drs-unable-to-migrate.png)

Trying to manually migrate VMs on any host also produced the following error during pre-validation, before you could even finish the migration wizard:

```
vMotion pre-validation error:  "Error occurred when validating selection. Check the log files for details."
```

It didn't immediately occur to me that these errors could also be related to profile-driven storage.  We don't use vVOLs or vSAN in this environment and I rarely think about or look at storage policies.  After poking around for a bit, on a whim I decided to restart the profile-driven storage service on this vCenter as well and sure enough it fixed the problem.  For future reference I'll keep in mind that profile-driven storage is integral to many vCenter operations whether you actively use storage policies or not.

It may just be coincidence, but the day before all of this happened an external PSC had become unresponsive and had to be restarted.  Both vCenters point to this external PSC.  It seems odd that both vCenters had an issue with profile-driven storage at the same time, but I don't really know if the problem with their PSC had anything to do with it.

[1]: https://kb.vmware.com/s/article/2118557
[2]: https://kb.vmware.com/s/article/53707
