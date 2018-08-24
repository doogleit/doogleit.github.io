---
layout: single
title:  "vRealize Suite Lifecycle Manager 1.3 Overview"
date:   2018-08-23 19:07:19 -0500
categories: vrealize
---

This is a quick overview of vRealize Suite Lifecycle Manager and some of the amazing automation and lifecycle management capabilities that it has to offer for the vRealize Suite. vRSLCM was [initially released last fall][1], providing a simplified means to deploy and upgrade a vRealize environment.  You can import an existing environment or define the required configuration up front and deploy the vRealize Suite in an automated fashion to an existing Datacenter/vCenter cluster. Version 1.2 was [released this past spring][2] and added some big features like content management and integration with the [VMware VSX marketplace][3].  The content management functionality allows IT teams to adopt a DevOps approach with source control integration, versioning, and content pipelines.  Finally, version 1.3 was [just released this summer][4], adding support for vRealize Network Insight, content management for vSphere templates along with vSphere Content Library integration, and a pre-validation check for upgrades.  

I'm excited to explore the content management features some more when I have the spare cycles.  For now I'm going to highlight some of the features it has for upgrading the vRealize Suite and how easy the upgrade process can be for a large vRealize environment.  I recently used vRSLCM to upgrade a decent sized vRealize environment and overall it went fairly smoothly.  For an overview on getting vRSLCM initially installed and setup checkout [this article at TheHumbleLab][5] or [this one at dutchvblog][6].

One of the first things I liked about vRSLCM is that you can export the configuration of each product to a JSON file and save it in version control or import it to deploy the same configuration to another environment.  There is also an option to "Create Snapshot" which is handy for starting an upgrade.  One click will snapshot all of the VMs in that environment, which can be a significant effort for a large vRA or vROps deployment.

![vRSLCM Menu](/assets/images/vrlcm-vra-menu.png)

The upgrade precheck is a really nice feature that was added in 1.3 and you can see below that the precheck for vRA is quite thorough.  This vRA environment consists of 6 VMs (2 appliances and 4 IaaS servers) and you can see below that it had 20 pages of results with 97 items in total.  All of the prerequisites for the IaaS servers were included!

![vRA Upgrade Precheck](/assets/images/vrlcm-vra-precheck.png)

With snapshots taken and the precheck passed, it's now just a matter of waiting or watching it run through the upgrade steps.  Under the Requests tab it walks through each of the steps it performs, including unregistering vRB and re-registering it after vRA is upgraded.

![vRA Upgrade Complete](/assets/images/vrlcm-vra-upgrade.png)

One thing I forgot to capture in the above vRA upgrade is the compatibility matrix.  The first step in the upgrade will check VMware's online compatibility matrix for you.  Below I'm upgrading vRealize Log Insight and it has checked the compatibility of the version I selected, listing compatible and incompatible versions of the other products I have installed.

![vRLI Compatability Matrix](/assets/images/vrlcm-vrli-compat.png)

Here is the vRB upgrade in progress.  Again it performs the unregistering and registering with vRA.

![vRB Upgrade Progress](/assets/images/vrlcm-vrb-progress.png)

Finally, vRB completed.

![vRB Upgrade Complete](/assets/images/vrlcm-vrb-complete.png)

Overall this greatly simplified my upgrade process and reduced the manual effort considerably.  I'm planning to upgrade vROps soon, but with the many changes in 6.7 I have a few dashboards/views that I still need to update.  I highly recommend taking a look at the [Upgrade Assessment Tool for vRealize Operations Manager 6.7][7] priot to upgrading.  It will give you a detailed report of any components and metrics that may be impacted.


[1]: https://blogs.vmware.com/management/2017/09/vrealize-suite-lifecycle-management.html
[2]: https://blogs.vmware.com/management/2018/03/vrealize-suite-lifecycle-manager-1-2-introducing-content-management-integrated-marketplace.html
[3]: https://marketplace.vmware.com/vsx/
[4]: https://blogs.vmware.com/management/2018/07/vrealize-suite-lifecycle-manager-1-3-whats-new.html
[5]: https://www.thehumblelab.com/deploying-vrealize-lifecycle-manager/
[6]: http://www.dutchvblog.com/vrealize-suite/working-with-vrealize-suite-lifecycle-manager/
[7]: https://kb.vmware.com/kb/53545
