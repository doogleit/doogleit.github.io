---
layout: single
title:  "EsxCli Awesomeness"
date:   2017-12-28 16:43:32 -0500
categories: vsphere esxi powercli
---
I recently wanted to know what the default setting was for a few of the ESXi advanced settings.  Since there's no telling what has been changed on many of my hosts, I was preparing to install a new virtual/nested instance of ESXi to check the defaults when I stumbled across this VMware vSphere Blog article on [Identifying Non-Default Advanced & Kernel Settings Using ESXCLI][vmware-blog].  To my suprise, esxcli has shown this info since version 5.1 and even has a '-delta' switch to show you exactly which settings have been changed!  

The esxcli list command shows additional information not available in vSphere Web Client or newer HTML5 Client, such as the default value for each setting and the min/max values for settings that take integers:

{% highlight shell %}
[root@esx001:~] esxcli system settings advanced list
   ...
   Path: /UserVars/ESXiShellTimeOut
   Type: integer
   Int Value: 0
   Default Int Value: 0
   Min Value: 0
   Max Value: 86400
   String Value:
   Default String Value:
   Valid Characters:
   Description: Time before automatically disabling local and remote shell access (in seconds, 0 disables).  Takes effect after the services are restarted.
   ...
{% endhighlight %}

The min/max values used to be shown in the old C# client, but I haven't seen this information in the web clients.  Appending the '--delta' switch (or abbreviated '-d') will list only the ESXi advanced settings that have been changed from their system defaults:

{% highlight shell %}
[root@esx001:~] esxcli system settings advanced list -d
{% endhighlight %}

You can also check specific settings by name:

{% highlight shell %}
[root@esx001:~] esxcli system settings advanced -o '/UserVars/ProductLockerLocation'
   Path: /UserVars/ProductLockerLocation
   Type: string
   Int Value: 0
   Default Int Value: 0
   Min Value: 0
   Max Value: 0
   String Value: /locker/packages/6.0.0
   Default String Value: /locker/packages/6.0.0
   Valid Characters: *
   Description: Path to VMware Tools repository
{% endhighlight %}

If you're a fan of PowerCLI like me, you can use Get-EsxCli in a similar manner.  This will list all the settings for the host:

{% highlight shell %}
PS C:\> (Get-EsxCli -VMHost 'esx001').system.settings.advanced.list()
{% endhighlight %}

And this will list only the deltas:

{% highlight shell %}
PS C:\> (Get-EsxCli -VMHost 'esx001').system.settings.advanced.list($true)
{% endhighlight %}

The first argument to the list method above is a boolean (true/false) indicating whether you only want the deltas. The second argument can be used to get a specific setting:

{% highlight shell %}
PS C:\> (Get-EsxCli -VMHost 'esx001').system.settings.advanced.list($null, '/UserVars/ProductLockerLocation')
{% endhighlight %}

This is the same as using the '--option' or '-o' switch with esxcli.  Lastly, it also accepts a third argument to get only the settings under a particular tree.  This is the same as using the '--tree' or '-t' switch with esxcli:

{% highlight shell %}
PS C:\> (Get-EsxCli -VMHost 'esx001').system.settings.advanced.list($null, $null, '/UserVars')
{% endhighlight %}

In summary, esxcli might be a lot more versatile than you think.  The delta switch is handy for auditing your configuration or just spotting settings that have been changed from the defaults, and can be especially useful if you have a lot of hosts and are trying to identify configuration drift.

[vmware-blog]: https://blogs.vmware.com/vsphere/2012/09/identifying-non-default-advanced-kernel-settings-using-esxcli-5-1.html
