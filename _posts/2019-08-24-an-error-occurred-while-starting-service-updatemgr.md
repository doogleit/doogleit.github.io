---
title:  "An error occurred while starting service 'updatemgr'"
categories: vSphere
tags: vum
---
I recently upgraded an environment to vSphere 6.7 Update 1.  After updating vCenter I updated a number of hosts with vSphere Update Manager before running into this problem.  During the remediation pre-check it would just spin its wheels until it eventually timed out.  I tested this on several hosts and it was consistent across all of them.  This pre-check error was occurring on all of the remaining ESXi 6.5 hosts in vCenter. When it did eventually time out I would see this very long error message.

>Waiting for task (vim.TaskInfo) { dynamicType = null, dynamicProperty = null, key = task-116427, task = ManagedObjectReference: type = Task, value = task-116427, serverGuid = b0879725-bf73-4114-8373-59429bae84d1, description = null, name = null, descriptionId = com.vmware.vcIntegrity.RemediatePrecheckTask, entity = ManagedObjectReference: type = HostSystem, value = host-704, serverGuid = b0879725-bf73-4114-8373-59429bae84d1, entityName = esx01.local.com, locked = null, state = queued, cancelled = false, cancelable = true, error = null, result = null, progress = null, reason = (vim.TaskReasonUser) { dynamicType = null, dynamicProperty = null, userName = administrator@vsphere.local }, queueTime = java.util.GregorianCalendar[time=?, areFieldsSet=false, areAllFieldsSet=true, lenient=true, zone=sun.util.calendar.ZoneInfo[id="GMT+00:00", offset=0, dstSavings=0, useDaylight=false, transitions=0, lastRule=null], firstDayOfWeek=1, minimalDaysInFirstWeek=1, ERA=1, YEAR=2019, MONTH=7, WEEK_OF_YEAR=1, WEEK_OF_MONTH=1, DAY_OF_MONTH=12, DAY_OF_YEAR=1, DAY_OF_WEEK=5, DAY_OF_WEEK_IN_MONTH=1, AM_PM=0, HOUR=0, HOUR_OF_DAY=15, MINUTE=3, SECOND=9, MILLISECOND=791, ZONE_OFFSET=0, DST_OFFSET=0], startTime = null, completeTime = null, eventChainId = 1511731, changeTag = null, parentTaskKey = null, rootTaskKey = null, activationId = com.vmware.vcIntegrity.RemediatePrecheckTask } took more than 300 seconds! Try using timeout for longer tasks!

It seemed reasonable to try restarting the Update Manager service.  Unfortunately, after stopping the service it would not start back up.
```shell
root@vcenter [ ~ ]# service-control --stop vmware-updatemgr
Operation not cancellable. Please wait for it to finish...
Performing stop operation on service updatemgr...
Successfully stopped service updatemgr
root@vcenter [ ~ ]# service-control --start vmware-updatemgr
Operation not cancellable. Please wait for it to finish...
Performing start operation on service updatemgr...
Error executing start on service updatemgr. Details {
    "detail": [
        {
            "args": [
                "updatemgr"
            ],
            "localized": "An error occurred while starting service 'updatemgr'",
            "translatable": "An error occurred while starting service '%(0)s'",
            "id": "install.ciscommon.service.failstart"
        }
    ],
    "componentKey": null,
    "problemId": null,
    "resolution": null
}
Service-control failed. Error: {
    "detail": [
        {
            "args": [
                "updatemgr"
            ],
            "localized": "An error occurred while starting service 'updatemgr'",
            "translatable": "An error occurred while starting service '%(0)s'",
            "id": "install.ciscommon.service.failstart"
        }
    ],
    "componentKey": null,
    "problemId": null,
    "resolution": null
}
```

With a little searching I found this KB article, [Resetting VMware Update Manager Database on a vCenter Server Appliance 6.5 or vCenter Server Appliance 6.7 (2147284)][1].  While this might seem like a bit of a sledgehammer approach, there wasn't much to lose by reseting the DB.  I would only need to recreate the ESXi 6.7 image and a few baselines.  There wasn't a lot of configuration stored in Update Manager in this case.  So I proceeded to follow the article and found there were a few Python modules missing that required me to run some additional commands.

On the first try, running the reset-db command indicated the ipaddr module was missing:
```shell
root@vcenter [ ~ ]# /usr/lib/vmware-updatemgr/bin/updatemgr-utility.py reset-db
Traceback (most recent call last):
  File "/usr/lib/vmware-updatemgr/bin/updatemgr-utility.py", line 11, in <module>
    from vumUtils import registerWithVc, resetDB, manageConfig
  File "/usr/lib/vmware-updatemgr/bin/vumUtils.py", line 20, in <module>
    from cis.tools import get_install_parameter
  File "/usr/lib/vmware/site-packages/cis/tools.py", line 29, in <module>
    from cis.utils import *
  File "/usr/lib/vmware/site-packages/cis/utils.py", line 45, in <module>
    import ipaddr
ImportError: No module named ipaddr
```

Using pip, the [Python package manager][2], I installed the missing module.  Before running any of the below commands make sure you take a snapshot of vCenter and have a working backup. This may not be supported, so take appropriate precautions and proceed at your own risk.
```shell
root@vcenter [ ~ ]# pip install ipaddr
Collecting ipaddr
  Downloading https://files.pythonhosted.org/packages/9d/a7/1b39a16cb90dfe491f57e1cab3103a15d4e8dd9a150872744f531b1106c1/ipaddr-2.2.0.tar.gz
Installing collected packages: ipaddr
  Running setup.py install for ipaddr ... done
Successfully installed ipaddr-2.2.0
You are using pip version 8.1.2, however version 19.2.2 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
```

Trying to run reset-db again produced another missing module.
```shell
root@vcenter [ ~ ]# /usr/lib/vmware-updatemgr/bin/updatemgr-utility.py reset-db
Traceback (most recent call last):
  File "/usr/lib/vmware-updatemgr/bin/updatemgr-utility.py", line 11, in <module>
    from vumUtils import registerWithVc, resetDB, manageConfig
  File "/usr/lib/vmware-updatemgr/bin/vumUtils.py", line 22, in <module>
    from cis.cisreglib import SsoClient
  File "/usr/lib/vmware/site-packages/cis/cisreglib.py", line 32, in <module>
    from pyVim import sso
  File "/usr/lib/vmware/site-packages/pyVim/sso.py", line 27, in <module>
    from lxml import etree
ImportError: No module named lxml
```

Installing lxml revealed that yet another module was needed.
```shell
root@vcenter [ ~ ]# pip install lxml
Collecting lxml
  Downloading https://files.pythonhosted.org/packages/e4/f4/65d145cd6917131826050b0479be35aaccba2847b7f80fc4afc6bec6616b/lxml-4.4.1-cp27-cp27mu-manylinux1_x86_64.whl (5.7MB)
    100% |████████████████████████████████| 5.7MB 119kB/s
Installing collected packages: lxml
Successfully installed lxml-4.4.1
You are using pip version 8.1.2, however version 19.2.2 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.

root@vcenter [ ~ ]# /usr/lib/vmware-updatemgr/bin/updatemgr-utility.py reset-db
Traceback (most recent call last):
  File "/usr/lib/vmware-updatemgr/bin/updatemgr-utility.py", line 11, in <module>
    from vumUtils import registerWithVc, resetDB, manageConfig
  File "/usr/lib/vmware-updatemgr/bin/vumUtils.py", line 22, in <module>
    from cis.cisreglib import SsoClient
  File "/usr/lib/vmware/site-packages/cis/cisreglib.py", line 32, in <module>
    from pyVim import sso
  File "/usr/lib/vmware/site-packages/pyVim/sso.py", line 28, in <module>
    from OpenSSL import crypto
ImportError: No module named OpenSSL
```

This one was a bit trickier as there's no module named OpenSSL, but a quick search found that the module I was looking for was named pyopenssl.
```shell
root@vcenter [ ~ ]# pip install pyopenssl
Collecting pyopenssl
  Downloading https://files.pythonhosted.org/packages/01/c8/ceb170d81bd3941cbeb9940fc6cc2ef2ca4288d0ca8929ea4db5905d904d/pyOpenSSL-19.0.0-py2.py3-none-any.whl (53kB)
    100% |████████████████████████████████| 61kB 4.3MB/s
Collecting cryptography>=2.3 (from pyopenssl)
  Downloading https://files.pythonhosted.org/packages/e6/68/50698ce24c61db7d44d93a5043c621a0ca7839d4ef9dff913e6ab465fc92/cryptography-2.7-cp27-cp27mu-manylinux1_x86_64.whl (2.3MB)
    100% |████████████████████████████████| 2.3MB 401kB/s
Requirement already satisfied (use --upgrade to upgrade): six>=1.5.2 in /usr/lib/python2.7/site-packages (from pyopenssl)
Collecting enum34; python_version < "3" (from cryptography>=2.3->pyopenssl)
  Downloading https://files.pythonhosted.org/packages/c5/db/e56e6b4bbac7c4a06de1c50de6fe1ef3810018ae11732a50f15f62c7d050/enum34-1.1.6-py2-none-any.whl
Collecting asn1crypto>=0.21.0 (from cryptography>=2.3->pyopenssl)
  Downloading https://files.pythonhosted.org/packages/ea/cd/35485615f45f30a510576f1a56d1e0a7ad7bd8ab5ed7cdc600ef7cd06222/asn1crypto-0.24.0-py2.py3-none-any.whl (101kB)
    100% |████████████████████████████████| 102kB 10.4MB/s
Collecting cffi!=1.11.3,>=1.8 (from cryptography>=2.3->pyopenssl)
  Downloading https://files.pythonhosted.org/packages/8d/e9/0c8afd1579e5cf7bc0f06fbcd7cdb954cbc0baadd505973949a99337da1c/cffi-1.12.3-cp27-cp27mu-manylinux1_x86_64.whl (415kB)
    100% |████████████████████████████████| 419kB 2.5MB/s
Collecting ipaddress; python_version < "3" (from cryptography>=2.3->pyopenssl)
  Downloading https://files.pythonhosted.org/packages/fc/d0/7fc3a811e011d4b388be48a0e381db8d990042df54aa4ef4599a31d39853/ipaddress-1.0.22-py2.py3-none-any.whl
Collecting pycparser (from cffi!=1.11.3,>=1.8->cryptography>=2.3->pyopenssl)
  Downloading https://files.pythonhosted.org/packages/68/9e/49196946aee219aead1290e00d1e7fdeab8567783e83e1b9ab5585e6206a/pycparser-2.19.tar.gz (158kB)
    100% |████████████████████████████████| 163kB 7.4MB/s
Installing collected packages: enum34, asn1crypto, pycparser, cffi, ipaddress, cryptography, pyopenssl
  Running setup.py install for pycparser ... done
Successfully installed asn1crypto-0.24.0 cffi-1.12.3 cryptography-2.7 enum34-1.1.6 ipaddress-1.0.22 pycparser-2.19 pyopenssl-19.0.0
You are using pip version 8.1.2, however version 19.2.2 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
```

pyopenssl has a few dependencies which pip also graciously installed.  Finally the database reset script completed and I was able to remove the patch store and start the update manager service.
```shell
root@vcenter [ ~ ]# /usr/lib/vmware-updatemgr/bin/updatemgr-utility.py reset-db
Resetting vSphere Update Manager database...
Successfully reset vSphere Update Manager database
Start vSphere Update Manager service to apply the setting.
root@vcenter [ ~ ]# rm -rf /storage/updatemgr/patch-store/*
root@vcenter [ ~ ]# service-control --start vmware-updatemgr
Operation not cancellable. Please wait for it to finish...
Performing start operation on service updatemgr...
Successfully started service updatemgr
```

After recreating the ESXi 6.7 image and baseline I had no further issues with upgrading and patching the remaining hosts!  To summarize, the missing modules were ipaddr, lxml, and pyopenssl.

```shell
pip install ipaddr
pip install lxml
pip install pyopenssl
```

[1]: https://kb.vmware.com/s/article/2147284
[2]: https://en.wikipedia.org/wiki/Pip_(package_manager)