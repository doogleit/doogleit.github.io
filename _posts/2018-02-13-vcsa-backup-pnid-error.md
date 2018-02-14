---
layout: single
title:  "vCSA 6.5 Backup - PNID not resolved error"
date:   2018-02-13 17:52:03 -0500
categories: vcenter vcsa
---

The vCenter 6.5 appliance introduced a new, built-in backup process that can be accessed through the VAMI interface. On one of my vCenters the backup would run until it was nearly complete and then trow the error shown below.

![Backup Error](/assets/images/vcsa65-backup-error.png)

I double checked all the backup parameters and confirmed that I could backup another vCSA 6.5 appliance using the same parameters to the same destination.  I also confirmed that DNS resolution was working properly.  [VMware KB 2130599](https://kb.vmware.com/s/article/2130599) goes into some detail about the PNID (Primary Network Identifier).  After finding this article I suspected that the problem was due to the PNID being a short hostname and not the fully qualified domain name.  This particular appliance was migrated from a Windows vCenter using an early version of the migration tool, so I think the short name might have been carried over from Windows.

I decided to try adding a search domain to resolv.conf so that the domain name would be appended to DNS queries.  I logged into the appliance via SSH and appended a line to /etc/resolv.conf with the search domain.

{% highlight shell %}
nameserver 127.0.0.1
nameserver 10.0.0.1
nameserver 10.0.0.2
search mydomain.local
{% endhighlight %}

This didn't help, so I opened a support request with VMware.  After some additional troubleshooting around DNS resolution and research into the backup process they got back to me with a fairly simple work around.  In addition to having the search domain configured I needed to add a search option to one of the Python scripts for the backup process.

Make a backup copy of the Net.py file and edit it.
{% highlight shell %}
cd /usr/lib/applmgmt/backup_restore/py/vmware/appliance/backup_restore/util
cp Net.py Net.py.backup
vi Net.py
{% endhighlight %}

Changing the following line to add '+search'.
{% highlight python %}
# Original line commented out:
# ipV4Cmd = ['/usr/bin/dig', '+noedns', '+short', pnid, 'a']
# Modified with '+search':
ipV4Cmd = ['/usr/bin/dig', '+noedns', '+short', '+search', pnid, 'a'] 
{% endhighlight %}

Save the file and start a new backup.  Now the backup completes successfully!
