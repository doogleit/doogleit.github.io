---
layout: single
title:  "IPv6 DNS Error Upgrading to vCSA 6.5"
date:   2018-02-05 19:32:43 -0500
categories: vcenter vcsa
---
I was recently uprading a vCenter Server Appliance from 6.0 to 6.5 and ran into an interesting error during the upgrade pre-check.  It identified that the appliance's hostname had an IPv6 DNS record and indicated that IPv6 networking would not be preserved after the upgrade.

![Error Message](/assets/images/ipv6-dns-error.png)

I'm guessing that the IPv6 DNS entry was a leftover from Windows dynamically registering it because this vCenter was previously migrated from Windows to the vCSA. I knew that IPv6 was not being used and hoped there was a way to continue with the upgrade anyway, but this error does not allow you to proceed.  I planned to remove the IPv6 entry as the error message suggests, however deleting a DNS record can be a significant change in some environments.  So I began looking for a workaround that would allow me to proceed with the upgrade and fix DNS later.

First I disabled IPv6 in the DCUI on both the source and destination appliance.

![vCSA DCUI](/assets/images/vcsa-dcui-ipv6.png)

If you're unfamiliar with the upgrade process it actually deploys a new 6.5 appliance with a temporary IP address, migrates the configuration and data from the 6.0 appliance (the source) to the 6.5 appliance(the destination), powers off the 6.0 appliance, and finally the 6.5 appliance assumes the network identity (IP and hostname) of the old one.  With IPv6 disabled on both appliances I logged into them with SSH and verified that running "dig \`hostname\`" did not return any IPv6 addresses.

The error message persisted, but at this point I could be confident that IPv6 was not being used and I suspected that the pre-upgrade script was using a Python library that was still resolving the IPv6 address.  I should insert a warning here.  The following is purely for educational purposes. It should not be attempted in anything resembling a production environmentand and is most likely unsupported.

Running 'ps aux' in my SSH session on the 6.5 appliance revealed several upgrade scripts running in '/usr/lib/vmware/cis_upgrade_runner/bootstrap_scripts'.  I changed to this directory and began searching for the source of the error message.  Running "grep 'resolves to IPv6' *.py" directed me to the file "upgrade_commands.py", where I located the section that was checking for this condition and commented it out by prefixing each line with a "#".

{% highlight python %}
    #if hasNetConflicts:
    #    addrStr = ', '.join(ipv6Addresses)
    #    logger.error("Source hostname '%s' resolves to IPv6 address(es) '%s', but "
    #                 "IPv6 networking will not be preserved during the upgrade.",
    #                 sourceFqdn, addrStr)
    #    self.reporter.addError(
    #        text=_(IPV6_RESOLVABLE_IN_IPV4_NETCONFIG_ERROR_TEXT,
    #               [sourceFqdn, addrStr]),
    #        description=_(IPV6_RESOLVABLE_IN_IPV4_NETCONFIG_ERROR_DESCRIPTION,
    #                      [sourceFqdn, addrStr]),
    #        resolution=_(IPV6_RESOLVABLE_IN_IPV4_NETCONFIG_ERROR_RESOLUTION,
    #                     sourceFqdn))
{% endhighlight %}

After commenting out the above section that was logging the IPv6 error I restarted the upgrade and it completed without issue.  Again, I don't recommend doing this unless it is a true test environment that you can afford to rebuild if necessary.  I just wanted to document/share the experience and a few details about how the vCSA 6.5 pre-upgrade check works.  If you haven't tried out 6.5 yet I highly recommend it.  See more about what's new over on the [VMware vSphere Blog](https://blogs.vmware.com/vsphere/2016/10/whats-new-in-vsphere-6-5-vcenter-server.html).

