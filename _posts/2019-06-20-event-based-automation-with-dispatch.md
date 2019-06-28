---
title: "Event Based Automation with VMware Dispatch"
image: /assets/images/dispatch-logo.png
categories: vsphere
tags: automation dispatch
---

One of the things that immediately drew my attention to Dispatch was the vCenter event driver.  Executing an automated task based on a vCenter event firing has always been something really attractive to me.  There are other ways to do this, but it always seemed like they were just not ideal for the environment I was working with for one reason or another. Most of the examples that I've seen use a vCenter alarm to run a command or send an SNMP trap.  While these work well enough (I've been using alarm actions to run scripts for some time), there are a few potential issues to consider.  

* Alarm actions can be disabled.  It's too easy to disable alarm actions on an object in vCenter and forget about it.  For example, let's say I'm putting a host in maintenance mode so I disable alarm actions for it.  I don't want everyone to be alerted while I'm working on and rebooting a host that is in maintenance.  Unfortunately, this also disables any alarm actions that I'm using for automation.  It would actually be really cool to have alarms automatically enabled/disabled based on a host's maintenance mode status.  However, if I'm using alarm actions to trigger the automation, then once alarm actions are disabled they can't turn themselves back on!

* The alarm action to "run a command" may not be supported on the vCenter Server Appliance.  Installing anything on the vCenter Server Appliance is unsupported and highly discouraged, so running a custom script on the vCSA from an alarm action is probably not technically supported.  With that said, it's relatively unobtrusive to have a bash or Python script call an external REST API, such as a vRO workflow, when an alarm action is triggered on the vCSA. This is covered thoroughly in William Lam's article, [How to run a script from a vCenter alarm action in the VCSA?][5] Also if you like Slack, be sure to check out this cool write up on [vSphere Alarms with Slack and StackStorm][6].

* The alarm action to "send a notification trap" is sent to all SNMP receivers configured in vCenter.  You can't distinguish between sending a trap for a real alert/problem and sending one just to perform some automation.  Generally, this isn't a problem, but in a larger organization where you're sending all of your SNMP traps to an enterprise monitoring system it could be.  The monitoring team will need to filter out traps for alarms that are not really problems and only exist for the sake of automation.  Some common examples of this are when a new VM is created or when a host is placed into maintenance mode. For an example covering SNMP traps on VM creation see [Automatically Securing Virtual Machines Using vCenter Orchestrator][7].

This leads me to [VMware Dispatch][1] and its [vCenter event driver][4].  Dispatch runs on Kubernetes, so if you're like me and have been looking for something to try out Kubernetes with then here it is.  I haven't had a lot of opportunites to make use of containers while working in more traditional/conservative IT organizations and thought this would be a cool way to try out Kubernetes.  If you just want to get it up and running quickly, [Dispatch Solo][2] provides a quick-start option with all of the dependencies packaged in a single virtual appliance (OVA).  Their [quickstart page][3] covers deploying the OVA and a simple hello world example.  This is how I've started out and plan to transistion to the Knative branch on Kubernetes in the near future.

Another feature of Dispatch that I really like is built-in support for Powershell Core.  This makes it relatively easy to use existing Powershell/PowerCLI scripts and combine them with vCenter events.  A script that I've had scheduled for many years is a variation of Alans Renouf's original article, [Who Created That VM?][8], which sets the date a VM was created and the user that created it as custom attributes.  This has been a common topic over the years, also being covered [here][9], [here][10], and in various discussions on the VMTN community forums.  It's worth noting that the [VM creation date is now available on VMs created in vSphere 6.7][11], but you still might want to use one of the previous scripts if you're not completely on vSphere 6.7 yet and/or if you also want to know who created the VM.

Instead of scheduling the script to run periodically and hope that it finds all the new VMs since the last time it was run, I think it's more efficient to have it run automatically when a new VM is created.  The Dispatch documentation has an example for [hardening VMs in vSphere][12].  Most of this will be a repeat of that process, except that I'm going to use a script I modified for Dispatch to set the VM's "Created By" and "Created On" attributes.

Once you've deployed Dispatch you use the Dispatch CLI to manage it.  Dispatch CLI can be downloaded and installed on macOS and Linux. On Windows 10 you can use [Windows Subsystem for Linux][13] or if you deployed Dispatch Solo you can SSH to the appliance and just run the commands there.  I'm using Ubuntu on Windows Subsystem for Linux, but I've also SSH'd into the appliance in some cases and haven't had any issues.

Here is a brief outline of the process to setup a PowerCLI script with the vCenter event driver:
1. Create a Dispatch image with the dependencies we need (in this case VMware PowerCLI)
2. Create a secret with the vSphere credentials
3. Create a vCenter event driver using the previously created secret
4. Create a function from the PowerCLI script
5. Subscribe the new function to an event

Steps 1 - 3 only need to be performed once - as long as the dependencies and vCenter credentials don't change these items can be reused.  Steps 4 and 5 are repeated for each new function you create.  Instructions for creating the PowerCLI image, vCenter secret, and vCenter event driver are in the example, [Harden VMs in vSphere][12].  Those are prerequisites for the rest of this article.

One thing to note about the payload from the event driver is that not all event types contain the 'vm_name'.  The 'vm.deployed' event does (as demonstrated in the hardening VMs example), but the 'vm.created' event does not.  Here's an example of the input received from the event driver when a VM is deployed (i.e. deployed from template):

{% highlight json %}
{
    "category": "info",
    "message": "Template Windows 2016 deployed on host esx01.domain.com",
    "metadata": {
        "src_template": "Windows 2016",
        "vm_id": "VirtualMachine:vm-845219",
        "vm_name": "testvm01"
    },
    "time": "2019-06-07T17:26:29.951027Z"
}
{% endhighlight %}

The metadata section contains some useful info about the VM being deployed.  Here is an example when a VM is created (i.e. created from an OVA/OVF or wihtout a template):

{% highlight json %}
{
    "category": "info",
    "message": "Created virtual machine testvm02 on esx01.domain.com in Lab",
    "metadata": null,
    "time": "2019-06-07T17:30:57.93864Z"
}
{% endhighlight %}

The second example applies to most other event types, such as when a VM is cloned or a host enters maintenance mode.  

In the powershell script below I use the event time to retrieve events from vCenter at that time and then filter on the event message.  It then has all the event information available to it and can set the 'UserName' and 'CreatedTime' in the VM's 'Created By' and 'Created On' attributes.  The other important piece here is the 'handle' function, which sets some variables from the vCenter secret we provided and the input from the event driver.  Dispatch looks for this function and will run it by default.  You can also get this script from my [GitHub repository][14].

{% highlight powershell %}
function handle($context, $payload) {
	<#
	.SYNOPSIS
		Function to set custom attributes on a newly deployed VM.
	.DESCRIPTION
		The 'handle' function is called by a VMware Dispatch event subscription
		when a new VM is deployed to automatically set the custom attributes
		'Created On' and 'Created By' on the new VM.
	.LINK
		https://vmware.github.io/dispatch/documentation/examples/vcenter-events
	#>
	$username = $context.secrets.username
	$password = $context.secrets.password
	$hostname = $context.secrets.host
	$eventTime = [DateTime]$payload.time
	$eventMessage = $payload.message
	
	Import-Module vmware.vimautomation.core -verbose:$false
	Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false
	
	# Connect to vSphere
	Write-Host "Checking VC Connection is active"
	if (-not $global:defaultviservers) {
		Write-Host "Connecting to $hostname"
		$viserver = Connect-VIServer -server $hostname -User $username -Password $password
	}
	else {
		Write-Host "Already connected to $hostname"
	}
	
	# Get the event by filtering on time and message
	$eventManager = Get-View (Get-View ServiceInstance).Content.EventManager
	$eventFilterSpec = New-Object VMware.Vim.EventFilterSpec
	$EventFilterSpec.Time = New-Object VMware.Vim.EventFilterSpecByTime
	$EventFilterSpec.Time.beginTime = $eventTime
	$EventFilterSpec.Time.endTime = $eventTime.AddSeconds(1)
	$event = $eventManager.QueryEvents($EventFilterSpec) | Where-Object 'FullFormattedMessage' -eq $eventMessage
	
	if ($event) {
		$vm = Get-VM -Name $event.Vm.Name
		if ($event.UserName) {
			Write-Host "Setting 'Created By' = '$($event.UserName)' on VM $VM"
			Set-Annotation -Entity $VM -CustomAttribute "Created By" -Value $event.UserName | Out-Null
		}
		if ($event.CreatedTime) {
			Write-Host "Setting 'Created On' = '$($event.CreatedTime)' on VM $VM"
			Set-Annotation -Entity $VM -CustomAttribute "Created On" -Value $event.CreatedTime | Out-Null
		}
	}
	else {
		Write-Host "The event was not found."
	}
	
	return "success"
}
{% endhighlight %}

Save the above code to a file named 'Set-VMAttributes.ps1'.  Next we'll create a Dispatch function and verify that the function is ready.

{% highlight shell %}
$ dispatch create function set-vmattributes Set-VMAttributes.ps1 --image=powershell-powercli --secret vsphere
Created function: set-vmattributes
$ dispatch get function set-vmattributes
        NAME       |                       FUNCTIONIMAGE                       | STATUS |         CREATED DATE
-------------------------------------------------------------------------------------------------------------------
  set-vmattributes | dispatch/func-c8eb1e08-4c43-4697-b9bb-2902fb490c89:latest | READY  | Fri Jun  7 17:11:48 DST 2019
{% endhighlight %}

Finally, we'll create event subscriptions for our new function.  This ties the function we just created to the vCenter events when a VM is deployed, created, cloned, or registered.

{% highlight shell %}
$ dispatch create subscription --event-type vm.deployed set-vmattributes --name set-vmattributes-vmdeployed
created subscription: set-vmattributes-vmdeployed
$ dispatch create subscription --event-type vm.created set-vmattributes --name set-vmattributes-vmcreated
created subscription: set-vmattributes-vmcreated
$ dispatch create subscription --event-type vm.cloned set-vmattributes --name set-vmattributes-vmcloned
created subscription: set-vmattributes-vmcloned
$ dispatch create subscription --event-type vm.registered set-vmattributes --name set-vmattributes-vmregistered
created subscription: set-vmattributes-vmregistered
{% endhighlight %}

Run `dispatch get subscription` to confirm that the subscriptions are all ready.

{% highlight shell %}
$ dispatch get subscription
              NAME              |  EVENT TYPE   |  FUNCTION NAME   | STATUS |         CREATED DATE
------------------------------------------------------------------------------------------------------
  set-vmattributes-vmcloned     | vm.cloned     | set-vmattributes | READY  | Fri Jun  7 18:16:10 DST 2019
  set-vmattributes-vmcreated    | vm.created    | set-vmattributes | READY  | Fri Jun  7 18:16:02 DST 2019
  set-vmattributes-vmdeployed   | vm.deployed   | set-vmattributes | READY  | Fri Jun  7 18:15:53 DST 2019
  set-vmattributes-vmregistered | vm.registered | set-vmattributes | READY  | Fri Jun  7 18:16:18 DST 2019
{% endhighlight %}

Deploy or create a couple of VMs and confirm that the function has run:

{% highlight shell %}
$ dispatch get run set-vmattributes
                   ID                  |     FUNCTION     | STATUS |               STARTED               |              FINISHED
------------------------------------------------------------------------------------------------------------------------------------------
  16d3495e-0834-4bdc-a9c6-8bfc69a0c302 | set-vmattributes | READY  | 2019-06-07T18:40:00.423122825-04:00 | 2019-06-07T18:40:02.240076238-04:00
  6ae02fb0-9ca1-48ae-922c-fa6ccf9df876 | set-vmattributes | READY  | 2019-06-07T18:34:59.099968839-04:00 | 2019-06-07T18:35:07.330598053-04:00
{% endhighlight %}

To see all of the output from a particular run, copy the ID from the above list and use it in the command below.

{% highlight shell %}
$ dispatch get run set-vmattributes 16d3495e-0834-4bdc-a9c6-8bfc69a0c302 --json
{% endhighlight %}

This produces a lot of useful information for troubleshooting, such as the event type, the input to the function, and all of the function's stdout and stderr.

{% highlight json %}
{
    "event": {
        "eventType": "vm.deployed",
        "eventTypeVersion": "0.1",
        "cloudEventsVersion": "0.1",
        "source": "vcenter1",
        "eventID": "8a320cc1-4fe9-42bd-9e4d-5499fb826f34",
        "eventTime": "0001-01-01T00:00:00.000Z",
        "contentType": "application/json",
        "data": null
    },
    "executedTime": 1560534000423122825,
    "faasId": "8eb037f0-d06e-4add-bf2b-43f6ab2fb57f",
    "finishedTime": 1560534002240076238,
    "functionId": "39b72f36-aa87-47e6-80f3-dcc37bff1316",
    "functionName": "set-vmattributes",
    "input": {
        "category": "info",
        "message": "Template Windows 2016 deployed on host esx03.domain.com",
        "metadata": {
            "src_template": "Windows 2016",
            "vm_id": "VirtualMachine:vm-847587",
            "vm_name": "testvm04"
        },
        "time": "2019-06-07T18:40:00.004554Z"
    },
    "logs": {
        "stderr": null,
        "stdout": [
            "Checking VC Connection is active",
            "Already connected to vcenter.nghs.com",
            "VERBOSE: 06/07/2019 18:40:00\tGet-View\tFinished execution",
            "DEBUG: 06/07/2019 18:40:00\tGet-View\t",
            "VERBOSE: 06/07/2019 18:40:00\tGet-View\tFinished execution",
            "DEBUG: 06/07/2019 18:40:00\tGet-View\t",
            "VERBOSE: 06/07/2019 18:40:02\tGet-VM\tFinished execution",
            "DEBUG: 06/07/2019 18:40:02\tGet-VM\t",
            "Setting 'Created By' = 'administrator@vsphere.local' on VM testvm04",
            "DEBUG: 06/07/2019 18:40:02\tSet-Annotation\t5246050e-1666-5ef9-f687-464457e7abba\t",
            "VERBOSE: 06/07/2019 18:40:02\tSet-Annotation\tFinished execution",
            "DEBUG: 06/07/2019 18:40:02\tSet-Annotation\t",
            "Setting 'Created On' = '06/07/2019 18:40:00' on VM testvm04",
            "DEBUG: 06/07/2019 18:40:02\tSet-Annotation\t5246050e-1666-5ef9-f687-464457e7abba\t",
            "VERBOSE: 06/07/2019 18:40:02\tSet-Annotation\tFinished execution",
            "DEBUG: 06/07/2019 18:40:02\tSet-Annotation\t",
            ""
        ]
    },
    "name": "16d3495e-0834-4bdc-a9c6-8bfc69a0c302",
    "output": [
        [
            {
                "CEIPDataTransferProxyPolicy": 1,
                "DefaultVIServerMode": 1,
                "DisplayDeprecationWarnings": true,
                "InvalidCertificateAction": 4,
                "ParticipateInCEIP": null,
                "ProxyPolicy": 1,
                "Scope": 1,
                "VMConsoleWindowBrowser": null,
                "WebOperationTimeoutSeconds": 300
            },
            {
                "CEIPDataTransferProxyPolicy": null,
                "DefaultVIServerMode": null,
                "DisplayDeprecationWarnings": null,
                "InvalidCertificateAction": 4,
                "ParticipateInCEIP": null,
                "ProxyPolicy": null,
                "Scope": 2,
                "VMConsoleWindowBrowser": null,
                "WebOperationTimeoutSeconds": null
            },
            {
                "CEIPDataTransferProxyPolicy": null,
                "DefaultVIServerMode": null,
                "DisplayDeprecationWarnings": null,
                "InvalidCertificateAction": null,
                "ParticipateInCEIP": null,
                "ProxyPolicy": null,
                "Scope": 4,
                "VMConsoleWindowBrowser": null,
                "WebOperationTimeoutSeconds": null
            }
        ],
        "success"
    ],
    "reason": null,
    "secrets": [
        "vsphere"
    ],
    "services": null,
    "status": "READY",
    "tags": []
}
{% endhighlight %}

That's it!  Now whenever a VM is deployed Dispatch will automatically set these custom attributes on the VM.  My next endeavor is to run a function when hosts enter/exit maintenance mode.  Hopefully I will have an article on that soon!

[1]: https://vmware.github.io/dispatch/
[2]: https://github.com/vmware/dispatch/tree/solo
[3]: https://vmware.github.io/dispatch/documentation/front/quickstart
[4]: https://github.com/dispatchframework/dispatch-events-vcenter
[5]: https://www.virtuallyghetto.com/2016/06/how-to-run-a-script-from-a-vcenter-alarm-action-in-the-vcsa.html
[6]: https://www.greenreedtech.com/vsphere-alarms-with-slack-and-stackstorm/
[7]: https://blogs.vmware.com/vsphere/2012/07/automatically-securing-virtual-machines-using-vcenter-orchestrator.html
[8]: http://www.virtu-al.net/2010/02/23/who-created-that-vm/
[9]: https://psvmware.wordpress.com/2012/11/27/when-was-vm-created-how-vm-was-created-and-who-has-created-this-vm/
[10]: http://www.vmspot.com/a-very-simple-powercli-script-to-gather-vm-creation-dates/
[11]: https://www.virtuallyghetto.com/2018/04/vm-creation-date-now-available-in-vsphere-6-7.html
[12]: https://vmware.github.io/dispatch/documentation/examples/vcenter-events
[13]: https://docs.microsoft.com/en-us/windows/wsl/install-win10
[14]: https://github.com/doogleit/powercli-misc/tree/master/dispatch