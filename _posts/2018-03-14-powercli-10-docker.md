---
layout: single
title:  "VMware PowerCLI 10.0 on Docker Hub"
date:   2018-03-14 20:12:23 -0500
categories: automation powershell
---

Last week I wrote about the release of VMware PowerCLI 10.0 and running it on Photon OS.  After running through this exercise I thought it would be really useful to have a Docker container image with PowerCLI 10.0 already installed and published on Docker Hub.  VMware has had a [docker image for PowerCLI core][1] available for a while now, but it's not yet updated for PowerCLI 10.

For my docker container I'm using [Microsoft's PowerShell docker image][2], installing the VMware PowerCLI module, and doing some basic configuration.  It installs some other common tools for PowerCLI, including PowervRA, vCheck, VMware's Powershell sample scripts, and Vester.  I have to thank VMware for their open source efforts and the existing [PowerCLI Dockerfile][3] since I copied a section from it to download the sample scripts.  You can pull the PowerCLI 10.0 container image by running:

```
docker pull doogleit/powerclicore
```

Or [check it out on Docker Hub][4].  I'm hoping to start using containers for some of my PowerCLI work and would love to hear how others are planning to or already using it.

[1]: https://hub.docker.com/r/vmware/powerclicore/
[2]: https://hub.docker.com/r/microsoft/powershell/
[3]: https://hub.docker.com/r/vmware/powerclicore/~/dockerfile/
[4]: https://hub.docker.com/r/doogleit/powerclicore/
