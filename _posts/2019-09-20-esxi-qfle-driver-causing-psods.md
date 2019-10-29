---
title:  "ESXi QFLE3 Driver Causing PSODs"
date:   2019-10-29 15:08:43 -0400
categories: vSphere
tags: esxi psod
---

I had a couple of exciting months after upgrading all of my vSphere environments to 6.7 Update 1.  There are two separate bugs in the qfle3 drivers that cause PSODs. The first PSOD didn't occur until about a month after all the hosts were updated.

![qfle3 PSOD](/assets/images/qfle3-psod.png)


If the PSODs had started occuring sooner I think the issue would have been caught in a dev/test environment.  After the first one several more hosts crashed in the following week, making me think that this was like a ticking time bomb. It seemed like the longer the host was running the more likely it was to crash, but I don't have any definitive evidence of that.  VMware support was quick to identify the issue and provide a resolution, which was upgrading the qfle3 driver to [version 1.0.86][1].  This is relatively simple using update manager and you can double check the version using esxcli.

```shell
[root@esx01:~] esxcli software vib list | grep qfle
qfle3                          1.0.86.0-1OEM.670.0.0.8169922        QLC      VMwareCertified   2019-08-30
qfle3f                         1.0.63.0-1OEM.670.0.0.8169922        QLC      VMwareCertified   2019-08-12
qfle3i                         1.0.20.0-1OEM.670.0.0.8169922        QLC      VMwareCertified   2019-08-12
```

Not long after getting that updated driver rolled out we had another host crash with a slightly different PSOD.

![qfle3f PSOD](/assets/images/qfle3f-psod.png)

This one was caused by the qfle3f, or FCOE, driver.  According to [VMware KB 71361][2] a fix was due to be released soon and the workaround was to either disable the FCOE driver (if FCOE isn't in use) or switch to the bnx2x driver. [VMware KB 52044][3] may also be related to this and has the esxcli commands to disable qfle3 and enable bnx2x. 

To disable the qfle3f driver using esxcli run the following and reboot the host.
 
```shell
[root@esx01:~] esxcli system module set --enabled=false --module=qfle3f
```

To disable it on several hosts it is easier using PowerCLI. For example, the PowerCLI one-liner below will disable it on every host in "Cluster-01".  However you still need to reboot each host after running the command.

```powershell
get-cluster 'Cluster-01' | get-vmhost | % { (get-esxcli -vmhost $_).system.module.set($false, $null, "qfle3f") }
```

To check the status of the driver on each host, showing if the driver is enabled and loaded, I used this PowerCLI one-liner.

```powershell
get-cluster 'Cluster-01' | get-vmhost | % { $vmhost = $_.name; (get-esxcli -vmhost $_).system.module.list() |where name -eq qfle3f | select @{l='vmhost';e={$vmhost}}, Name, IsEnabled, IsLoaded }
```

To also check the qfle3 driver version, I used the following one-liner, which admittedly is pretty ugly and probably not optimal, but it allowed me to quickly check hundreds of hosts.

```powershell
get-vmhost | % { $vmhost = $_.name; $esxcli = get-esxcli -vmhost $_; $esxcli.system.module.list() |where name -eq qfle3f | select @{l='vmhost';e={$vmhost}}, Name, IsEnabled, IsLoaded, @{N="qfle3 version";E={$esxcli.system.module.get("qfle3").version}} } | ft -auto
```

There were a few hosts where I needed to use FCOE.  For these I enabled the bnx2x driver and disabled qfle3.

```shell
[root@esx01:~] esxcli system module set --enabled=true --module=bnx2x
[root@esx01:~] esxcli system module set --enabled=false --module=qfle3
```
After rebooting the host I can confirm that the bnx2 drivers are listed and the version expected.

```shell
[root@esx01:~] esxcli software vib list | grep bnx2
net-bnx2                       2.2.6b.v60.2-1OEM.600.0.0.2494585    QLogic   VMwareCertified   2018-11-13
net-bnx2x                      2.713.60.v60.1-1OEM.600.0.0.2494585  QLogic   VMwareCertified   2018-11-13
scsi-bnx2fc                    1.713.60.v60.1-1OEM.600.0.0.2494585  QLogic   VMwareCertified   2018-11-13
scsi-bnx2i                     2.713.60.v60.1-1OEM.600.0.0.2494585  QLogic   VMwareCertified   2018-11-13
```

To wrap up this post I want to mention that VMware Skyline is making some really great progress towards proactively identifying these types of problems.  I've since noticed that Skyline identifies and reports on findings that can cause a PSOD.  You can see in the screenshot below a couple more driver issues, one of them for the qfle3i (iSCSI) driver which can also cause a PSOD.

![Skyline Findings](/assets/images/skyline-psod-findings.png)

In the details of each finding you can see any recommendations and helpful links.

![Skyline Details](/assets/images/skyline-psod-details.png)

I'm looking forward to using Skyline to more proactively address these types of issues.  And now it looks like I have some more findings that need to be addressed!

[Update 10/29/2019] One quick note, since I originally wrote this [KB 71361][2] has been updated with the fixed qfle3f driver.  The workarounds I discussed should no longer be needed!

[1]: https://my.vmware.com/web/vmware/details?downloadGroup=DT-ESXI67-QLOGIC-QFLE3-10860&productId=742
[2]: https://kb.vmware.com/s/article/71361
[3]: https://kb.vmware.com/s/article/52044