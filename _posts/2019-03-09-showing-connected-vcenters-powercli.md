---
layout: single
title:  "Showing Connected vCenters in PowerCLI"
categories: vsphere
tags: vsphere powercli 
---
If you work with multiple vCenters in PowerCLI you might find it useful to show the vCenter connections in the Powershell title bar. I often have multiple Powershell windows open with each window connected to a different vCenter.  Sometimes one may be connected to multiple vCenters and keeping track of the different sessions can be a bit daunting.  By using a custom Prompt function in your Powershell profile you can set the title bar to show all of the vCenters that it's connected to. Here is the prompt function that I recently added to my Powershell profile:

{% highlight powershell %}
Function Prompt {
	"PS $($executionContext.SessionState.Path.CurrentLocation)$('>' * ($nestedPromptLevel + 1)) ";
	# .Link
	# http://go.microsoft.com/fwlink/?LinkID=225750
	# .ExternalHelp System.Management.Automation.dll-help.xml

	$host.UI.RawUI.WindowTitle = $global:DefaultVIServers.Name -join "|"
}
{% endhighlight %}

If you already have a custom prompt function just add the line that sets `$host.UI.RawUI.WindowTitle`.  The `$global:DefaultVIServers` variable is a PowerCLI variable that contains all of the vCenters that your session has connections to.  If there are multiple vCenters the example above joins them into a single string using a verticle bar `|` as the separator.  When you use PowerCLI's `Connect-VIServer` cmdlet to connect to a vCenter it adds it to the `$global:DefaultVIServers` variable.  There is also a singular version of this variable, `$global:DefaultVIServer`, which is the default server and is usually the last one you connected to.

This has been really helpful to me in keeping track of what vCenters different Powershell windows are connected to.

Now when you open Powershell the default window title will be blank.
![Powershell Title](/assets/images/psh-title.png)

Once you connect to a vCenter the name will be shown in the window title.
![Powershell Title](/assets/images/psh-title1.png)

Multiple vCenters will be separated by a verticle bar.
![Powershell Title](/assets/images/psh-title2.png)

For more info on the Prompt function run `Get-Help about_Prompts` in Powershell or see [Microsoft's online docs][1].  If you want to know more about Powershell profiles see `Get-Help about_Profiles`.

[1]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_prompts?view=powershell-6