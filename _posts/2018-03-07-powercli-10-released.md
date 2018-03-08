---
layout: single
title:  "VMware PowerCLI 10.0 Released!"
date:   2018-03-07 19:52:02 -0500
categories: automation powershell
---

VMware PowerCLI 10.0 was released last week. The most notable enhancement in this release is official multi-platform support for Mac OS and Linux.  Linux and Mac OS users rejoice!  The only pre-requisite is PowerShell Core 6.0.  Not all of the modules are supported yet, but a good chunk of them are:

* VMware.VimAutomation.Cis.Core
* VMware.VimAutomation.Common
* VMware.VimAutomation.Core
* VMware.VimAutomation.Nsxt
* VMware.VimAutomation.Vds
* VMware.VimAutomation.Vmc
* VMware.VimAutomation.Sdk
* VMware.VimAutomation.Storage
* VMware.VimAutomation.StorageUtility

Another major change is the default certificate handling.  Self-signed or invalid certificates will now produe an error instead of just a warning.  This behvior can be changed using the `Set-PowerCLIConfiguration` cmdlet.  For all the details on this release checkout the article on the [VMware PowerCLI Blog][1].

I thought I would try this out on VMware's Photon OS.  It turns out you really need PowerShell Core 6.0.1 to use the PowerCLI 10.0 modules.  The PowerShell Core package that's currently available for Photon OS in VMware's repository is only 6.0.0.  

I tried a couple of the methods listed on Microsoft's site for [Installing PowerShell Core on macOS and Linux][2], but had little success.  I was initially trying this on Photon OS 1.0.  VMware has a simple upgrade process that didn't take very long. If you also have an older version see [Upgrading to Photon OS 2.0][3].  The Linux AppImage will run on Photon OS 2.0 and the modules load, but Connect-VIServer throws an error.

```
connect-viserver : 3/7/18 1:28:39 PM    Connect-VIServer    The type initializer for 'System.Net.Http.CurlHandler' threw an exception.
```

When trying to install some of the packages from Microsoft's site I was getting errors about libcurl dependencies missing.  This error looks like it may be similar in nature.  I eventually decided to just use a docker container.  This is definitely the easiest way to try this out right now on Photon OS.  It's worth noting that VMware has had an unsupported version available for a while and William Lam written some excellent articles about this, such as [5 ways to run a PowerCLI script using the PowerCLI Docker Container][4].  VMware's image hasn't been updated for PowerCLI 10.0, so I'm using Microsoft's Powershell container image below. 

On a new Photon OS installation you'll first need to enable docker.  This command starts the docker daemon.
```
systemctl start docker
```

This will enable the docker daemon on boot up.
```
systemctl enable docker
```

Next simply pull (download) and run the container.
```
docker pull microsoft/powershell
docker run -it microsoft/powershell
```

You should now have a nice PS prompt. 
```
PowerShell v6.0.1
Copyright (c) Microsoft Corporation. All rights reserved.

https://aka.ms/pscore6-docs
Type 'help' to get help.

PS />
```

Install VMware.PowerCLI and set the `-InvalidCertificateAction` for the PowerCLI configuration.  `Connect-VIServer` will throw and error if you leave this unset.  It appears that the only valid options for this platform are "Fail or "Ignore", so I'm using "Ignore" for my lab environment.
```
PS /> Install-Module VMware.PowerCLI
PS /> Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false
```

I initially tried using "Warn" or "Prompt" and got a helpful error message.
```
connect-viserver : 3/7/18 4:02:46 PM    Connect-VIServer    The InvalidCertificateAction setting 'Prompt' is not supported on this platform. Use Set-PowerCLIConfiguration to set the value for the InvalidCertificateAction option to Fail or Ignore.
```

Now you should be able to run `Connect-VIServer` and start using PowerCLI 10.0 on Photon OS.

[1]: https://blogs.vmware.com/PowerCLI/2018/02/powercli-10.html
[2]: https://docs.microsoft.com/en-us/powershell/scripting/setup/installing-powershell-core-on-macos-and-linux?view=powershell-6
[3]: https://github.com/vmware/photon/wiki/Upgrading-to-Photon-OS-2.0
[4]: https://www.virtuallyghetto.com/2016/10/5-different-ways-to-run-powercli-script-using-powercli-core-docker-container.html
