---
title:  "Automating Host Maintenance with VMware Dispatch"
categories: vsphere
tags: automation dispatch
header:
  og_image: /assets/images/dispatch-logo.png
---

Following up on my previous post, [Event Based Automation with VMware Dispatch][1], this one will demonstrate using Dispatch to automate actions when a host enters or exits maintenance mode.  The Dispatch function we're going to create will disable alarm actions on the host when it enters maintenance mode and re-enable alarm actions when the host exits maintenance mode.

Here is the Powershell script to disable/enable alarm actions on a host in vCenter.

{% highlight powershell %}
function handle($context, $payload) {
	<#
	.SYNOPSIS
		Disables/enables vCenter alarm actions on a host.
	.DESCRIPTION
		The 'handle' function is called by a VMware Dispatch event subscription
		when a host enters/exits maintenance mode to disable/enable alarm actions 
		for the host.
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
	$events = $eventManager.QueryEvents($EventFilterSpec) | Where-Object 'FullFormattedMessage' -eq $eventMessage
	
	# For some reason there are always two "EnteredMaintenanceMode" events.  One has an 'Unknown user' and no related events.
	# We'll filter out that one and only use the event with a valid user and related events.
	$event = $events | Where-Object 'UserName' -ne 'Unknown user'
	
	if ($event) {
		$alarmManager = Get-View AlarmManager
		if ($event.GetType().Name -match "EnteredMaintenanceMode") {
			# Disable alarm actions on the host.  "$event.Host.Host" is the MoRef of the host.
			Write-Host "Disabling alarm actions on host $($event.Host.Name)"
			$alarmManager.EnableAlarmActions($event.Host.Host, $false)
		}
		else {
			# Enable alarm actions on the host.  "$event.Host.Host" is the MoRef of the host.
			Write-Host "Enabling alarm actions on host $($event.Host.Name)"
			$alarmManager.EnableAlarmActions($event.Host.Host, $true)
		}
	}
	else {
		Write-Host "The event was not found."
	}
	
	return "success"
}
{% endhighlight %}

This is also available on [Github][2].  Save the above code to a file named 'Set-HostAlarmActions.ps1' or grab it using curl:

{% highlight shell %}
$ curl -ko Set-HostAlarmActions.ps1 'https://raw.githubusercontent.com/doogleit/powercli-misc/master/dispatch/Set-HostAlarmActions.ps1'
{% endhighlight %}

This time to create the Dispatch function we're going to use a yaml file.  Dispatch allows you to define functions (and other things) in yaml files.  When you start working with more functions this is a convenient way to define them and it makes it easier to update a function while you are testing it and making frequent changes.  Here is the yaml file to define this function.

{% highlight yaml %}
kind: Function
name: set-hostalarmactions
sourcePath: 'Set-HostAlarmActions.ps1'
image: powershell-powercli
schema: {}
secrets:
  - vsphere
{% endhighlight %}

Now to create the function we just need to reference the yaml file.

{% highlight shell %}
$ dispatch create -f set-hostalarmactions.yml
Created Function: set-hostalarmactions
{% endhighlight %}

And to update the function we simply use `dispatch update`, referencing the same yaml file.

{% highlight shell %}
$ dispatch update -f set-hostalarmactions.yml
Updated Function: set-hostalarmactions
{% endhighlight %}

Once the function is created we just need to subscribe it to the events for a host entering and exiting maintenance mode.

{% highlight shell %}
$ dispatch create subscription --event-type entered.maintenance.mode set-hostalarmactions --name set-hostalarmactions-entermaint
created subscription: set-hostalarmactions-entermaint
$ dispatch create subscription --event-type exit.maintenance.mode set-hostalarmactions --name set-hostalarmactions-exitmaint
created subscription: set-hostalarmactions-exitmaint
{% endhighlight %}

You can have multiple functions in the same yaml file.  There's a more complete example in the [documentation on Functions][3].  You can also have different items in the same file, for example combining the function and subscriptions:

{% highlight yaml %}
kind: Function
name: set-hostalarmactions
sourcePath: 'Set-HostAlarmActions.ps1'
image: powershell-powercli
schema: {}
secrets:
  - vsphere
---
kind: Subscription
eventtype: entered.maintenance.mode
function: set-hostalarmactions
name: set-hostalarmactions-entermaint
---
kind: Subscription
eventtype: exit.maintenance.mode
function: set-hostalarmactions
name: set-hostalarmactions-exitmaint
{% endhighlight %}

Additionally, you can reference the yaml file over http.  It appears that the function file (Set-HostAlarmActions.ps1 in this case) does however need to be on the local filesystem, but this breaks the setup down to essentially two commands.  

1. Download the function file:

{% highlight shell %}
$ curl -ko Set-HostAlarmActions.ps1 'https://raw.githubusercontent.com/doogleit/powercli-misc/master/dispatch/Set-HostAlarmActions.ps1'
{% endhighlight %}

2. Create/update everything by referencing the YAML file:

{% highlight shell %}
$ dispatch create -f 'https://raw.githubusercontent.com/doogleit/powercli-misc/master/dispatch/set-hostalarmactions.yml'
{% endhighlight %}

The documentation on yaml files is a little sparse at the moment, but you can retrieve the yaml for an object in Dispatch like so:

{% highlight shell %}
$ dispatch get subscription -o yaml
- createdtime: 1560881481
  eventtype: entered.maintenance.mode
  function: set-hostalarmactions
  id: ""
  kind: Subscription
  modifiedtime: 1561405311
  name: set-hostalarmactions-entermaint
  secrets: []
  status: READY
  tags: []
- createdtime: 1561388377
  eventtype: exit.maintenance.mode
  function: set-hostalarmactions
  id: ""
  kind: Subscription
  modifiedtime: 1561405311
  name: set-hostalarmactions-exitmaint
  secrets: []
  status: READY
  tags: []
{% endhighlight %}

Using the output as a template, take the properties and values that you would normally specify on the command line and put those in your yaml file.  With a little trial and error you can figure out what's required.

I'm working on another function to do essentially the same thing with nodes in SolarWinds.  If you use SolarWinds for monitoring it will unmanage a host in Solarwinds when it enters maintenance mode and re-manage it when it exits maintenance.  Look for a part 2 to this post, hopefully coming soon!

[1]: https://doogleit.github.io/2019/07/event-based-automation-with-dispatch/
[2]: https://github.com/doogleit/powercli-misc/tree/master/dispatch
[3]: https://vmware.github.io/dispatch/documentation/usage/functions
