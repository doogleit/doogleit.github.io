---
layout: single
title:  "Alerting when vSphere HA VM Monitoring Restarts a VM"
date:   2018-05-14 20:22:18 -0500
categories: vsphere
---

vSphere HA VM monitoring is a feature that I haven't used a great deal.  I've always been hesitant to have a VM reset automatically based only on a heartbeat that is received from a guest process (VMware Tools).  I recently found a [VMware KB article][1] that explains testing the VM monitoring in detail, including the fact that it also does a disk I/O status check after the heartbeat fails.  This ensures that there is no disk activity for 120 seconds and helps prevent an active guest from accidently being restarted.  Maybe it's just because I've never delved deeper into this, but I'm a little surprised I had not learned of this before now.  It makes an admin like me feel a lot more comfortable about using this feature and not having to worry about false positives or unecessary restarts.  I feel like this feature could have been better publicized, although to VMware's credit, there is a pretty good [blog post on this subject][2] that is now several years old.

VM monitoring can be enabled in the cluster's vSphere HA configuration.
![HA Settings](/assets/images/ha-vm-monitoring.png)

And there's a built-in alarm in vCenter that can alert you when a VM is restarted.
![vCenter Alarm](/assets/images/ha-vm-vcenter-alarm.png)

But what if you're using vROps to centralize all your alerts, or want to forward the alert to a REST API to do some cool automation?  This is where the power and flexibility of Log Insight comes in.  Log Insight makes it pretty easy to send an alert to vROps or to a webhook URL.  You might want to check out the [webhook-shims project on Github][3] if you're interested in sending alerts to another system's REST API.  It has some examples of using webhooks with a variety of other systems.

The first thing we want to figure out is how to setup the query in Log Insight.  It helps if there is an existing event to look for in vCenter.  Here you can look for the column "Event Type ID" to get an idea of the event to filter on in Log Insight.

![vCenter Event](/assets/images/ha-vm-vcenter-event.png)

After doing some queries in Log Insight I found that the event type I wanted to search on was 'com.vmware.vim25.VmdDsBeingResetWithScreenshotEvent'.  Just to be sure that I captured any VM reset event, I used a wildcard in my actual filter for the alert:
![Event Filter](/assets/images/ha-vm-event-filter.png)

In this case I was just sending it to vROps, so I just checked the box to "Send to vRealize Operations Manager" in the Notify settings.  

Finally we need to test the alert.  The [VMware KB article][1] describes how you can disable the disk I/O check by setting the advanced vSphere HA setting "das.iostatsInterval" to "0" (zero).  Once the disk I/O check is disabled you can kill the "vmtoolsd" process in the guest OS to simulate a VMware Tools heartbeat failure. However, if there are any production VMs running in the cluster you probably don't want to disable the disk I/O check.  I started investigating a way to stop all disk I/O on a test VM, but during the process I found that a minimal CentOS installation with the vmnic disconnected generates practically no disk I/O.  I killed the vmtoolsd process on it, left it alone, and HA rebooted it after a few minutes while also triggering the alert.  The Linux distro that you use for the test VM probably doesn't matter as long as it is a minimal installation that won't have any activity.  This minimal test VM made it really easy to test the VM monitoring and associated alerts without having to change the cluster's advanced settings and disable the disk I/O check.  Note that the 'das.iostatsInterval' setting can also be adjusted to more or less than the default 120 seconds for more or less agressive monitoring.  It's recommended to set this to the same as the failure interval in the VM monitoring settings for the cluster.  See the [VM and Application Monitoring section][4] in the HA Deepdive book for another excellent resource on this.  Now that I know more about how this feature works I'll be looking for opportunities to use it more in the future!


[1]: https://kb.vmware.com/s/article/1007307
[2]: https://blogs.vmware.com/vsphere/2014/03/vsphere-ha-vm-monitoring-back-basics.html
[3]: https://github.com/vmw-loginsight/webhook-shims
[4]: https://ha.yellow-bricks.com/vm_and_application_monitoring.html

