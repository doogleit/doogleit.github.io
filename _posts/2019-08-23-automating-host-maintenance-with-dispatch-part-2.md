---
title:  "Automating Host Maintenance with VMware Dispatch Part 2"
categories: vsphere
tags: automation dispatch solarwinds
header:
  og_image: /assets/images/stack.yml.png
---

For part 2 of [Automating Host Maintenance with VMware Dispatch][1] we're going to create a function that will unmanage the node in Solarwinds when a host enters maintenance mode and re-manage it when it exits maintenance.  In SolarWinds unmanaging a node is similar to putting an object in maintenance in vROps.  It stops polling that node and no new alerts will trigger.  An alternative is to simply mute alerts, which continues polling the node/collecting statistics, but mutes new alerts.  If you'd prefer to only mute alerts I have an example of how to do that commented out in the script.  However with only alerts muted the node will still turn red on dashboards and show that it is down if it is rebooted.  If you have a NOC or other team monitoring for things like this it is probably preferable to unmanage it.

First, we need to create a secret that Dispatch can use to access SolarWinds. The SolarWinds API documentation refers to it as the [SolarWinds Information Service (SWIS)][2], so I will be using the "SWIS" acronym in the examples below.  Create a json file that looks something like this.

{% highlight json %}
{
  "swishost": "swishost.domain.com",
  "swisuser": "<username>",
  "swispass": "<password>"
}
{% endhighlight %}

Fill in the values for your environment and save this to a file named swis.json.  Then create the secret.

{% highlight shell %}
$ dispatch create secret swis swis.json
Created secret: swis
{% endhighlight %}

Next download the Powershell script.

{% highlight shell %}
curl -ko Set-SwisNodeState.ps1 'https://raw.githubusercontent.com/doogleit/powercli-misc/master/dispatch/Set-SwisNodeState.ps1'
{% endhighlight %}

I had originally planned to use the SolarWinds Powershell module, [SwisPowerShell][3], however it isn't compatible with Powershell Core, so it won't work in the powershell-base image in Dispatch.  Instead I'm using the [SWIS REST API][5] below and the native Powershell cmdlet "Invoke-RESTMethod".  This Powershell script is similar to the one used previously to disable/enable host alarm actions.  I'm just going to highlight a few of the key sections below.

{% highlight powershell %}
	# Get variables from the Swis secret
	$swishost = $context.secrets.swishost
	$swisuser = $context.secrets.swisuser
	$swispass = $context.secrets.swispass
	
	...
	
		# Get the Swis node with a matching hostname
		# NOTE: DisplayName in SolarWinds needs to match hostname in vCenter
		$response = Invoke-RESTMethod -Credential $swiscred -SkipCertificateCheck -Uri "https://$($swishost):17778/SolarWinds/InformationService/v3/Json/Query?query=SELECT+NodeId,+Uri+FROM+Orion.Nodes+WHERE+DisplayName='$($event.Host.Name)'"
		$node = $response.results

	...

			if ($event.GetType().Name -match "EnteredMaintenanceMode") {
				# Unmanage the node
				Write-Host "Unmanaging node $($event.Host.Name)"
				$now = [DateTime]::UtcNow
				$until = $now.AddDays(365)
				Invoke-RESTMethod -Credential $swiscred -Method Post -ContentType "application/json" -SkipCertificateCheck -Uri "https://$($swishost):17778/SolarWinds/InformationService/v3/Json/Invoke/Orion.Nodes/Unmanage" -Body "[`"N:$nodeId`", `"$now`", `"$until`", `"false`"]"
				# To supress/mute alerts instead use:
				#Invoke-RESTMethod -Credential $swiscred -Method Post -ContentType "application/json" -SkipCertificateCheck -Uri "https://$($swishost):17778/SolarWinds/InformationService/v3/Json/Invoke/Orion.AlertSuppression/SuppressAlerts" -Body "[[`"$nodeUri`"]]"
			}
			else {
				# Remanage the node
				Write-Host "Remanaging node $($event.Host.Name)"
				Invoke-RESTMethod -Credential $swiscred -Method Post -ContentType "application/json" -SkipCertificateCheck -Uri "https://$($swishost):17778/SolarWinds/InformationService/v3/Json/Invoke/Orion.Nodes/Remanage" -Body "[`"N:$nodeId`"]"
				# To resume alerts instead use:
				#Invoke-RESTMethod -Credential $swiscred -Method Post -ContentType "application/json" -SkipCertificateCheck -Uri "https://$($swishost):17778/SolarWinds/InformationService/v3/Json/Invoke/Orion.AlertSuppression/ResumeAlerts" -Body "[[`"$nodeUri`"]]"
			}
{% endhighlight %}

To only mute/suppress alerts comment out the first `Invoke-RESTMethod` in each section and uncomment the second one.  An advantage of using the REST API is that there's no additional modules needed.  We can use the same powershell-powercli image that I used in [previous articles][6] and that is created in the [Dispatch example documentation][7].

We'll use a YAML file to describe the function and two subscriptions for the events when hosts enter and exit maintenance mode.

{% highlight yaml %}
kind: Function
name: set-swisnodestate
sourcePath: 'Set-SwisNodeState.ps1'
image: powercli-swis
schema: {}
secrets:
  - vsphere
  - swis
---
kind: Subscription
eventtype: entered.maintenance.mode
function: set-swisnodestate
name: set-swisnodestate-entermaint
---
kind: Subscription
eventtype: exit.maintenance.mode
function: set-swisnodestate
name: set-swisnodestate-exitmaint
{% endhighlight %}

Now providing the YAML file to `dispatch create` will create the function and subscriptions.

{% highlight shell %}
$ dispatch create -f 'https://raw.githubusercontent.com/doogleit/powercli-misc/master/dispatch/set-swisnodestate.yml'
Created Function: set-swisnodestate
Created Subscription: set-swisnodestate-entermaint
Created Subscription: set-swisnodestate-exitmaint
{% endhighlight %}

Having most of the configuration in a YAML file like this also makes the cleanup rather easy.  If you decide not to use this function `dispatch delete` will happily accept the same file and delete everything for you.
{% highlight shell %}
$ dispatch delete -f 'https://raw.githubusercontent.com/doogleit/powercli-misc/master/dispatch/set-swisnodestate.yml'
Deleted Function: set-swisnodestate
Deleted Subscription: set-swisnodestate-entermaint
Deleted subscription: set-swisnodestate-exitmaint
{% endhighlight %}

And finally if you'd like to make any changes, download a local copy to modify and simply run `dispatch update`:

{% highlight shell %}
$ dispatch update -f set-swisnodestate.yml
Updated Function: set-swisnodestate
Updated Subscription: set-swisnodestate-entermaint
Updated Subscription: set-swisnodestate-exitmaint
{% endhighlight %}

One final note - it's also possible to have secrets defined in a yaml file.

{% highlight yaml %}
kind: Secret
name: swis
secrets:
  swishost: swishost.domain.com
  swisuser: <username>
  swispass: <password>
{% endhighlight %}

Generally speaking it's not a good idea to store credentials in plain text.  The json file above has the same problem, but you can delete the "swis.json" file after the secret is created in Dispatch.  For this reason I'm keeping the secret separate, ideally stored in a secure location away from the rest of the configuration.

That wraps up today's post.  I'm really enjoying some of the new posibilities with Dispatch.  If you're interested, here are a few references that I used for the SolarWinds API.

* [About the SolarWinds Information Service][2]
* [SWIS REST/JSON Endpoint][5]
* [Using the SolarWinds Information Service from PowerShell][3]
* [Orion.AlertSuppression][4]

[1]: https://doogleit.github.io/2019/07/automating-host-maintenance-with-dispatch/
[2]: https://github.com/solarwinds/OrionSDK/wiki/About-SWIS
[3]: https://github.com/solarwinds/OrionSDK/wiki/PowerShell
[4]: https://github.com/solarwinds/OrionSDK/wiki/Alerts#orionalertsuppression
[5]: https://github.com/solarwinds/OrionSDK/wiki/REST
[6]: https://doogleit.github.io/tags/#dispatch
[7]: https://vmware.github.io/dispatch/documentation/examples/vcenter-events
