---
layout: single
title:  "Posting Alerts to Microsoft Teams"
date:   2018-03-27 19:36:21 -0500
categories: automation
---

Webhooks provide a convenient way to send data to other systems.  VMware vRealize Operations and Log Insight both allow you to send alerts to a webhook, which is basically just sending a HTTP POST to an incoming webhook URL on the receiving system.  The receiving system usually expects a certain format and for this reason VMware has a [cool project on Github][1] that provides a number of webhook shims for translating the format between systems.  This allows alerts to be sent to a variety of collaboration platforms, ticketing systems, and other endpoints like vRealize Orchestrator.  I've only recently begun to realize the power of these.  This opens a new world of possiblities when you can trigger a remediation workflow based on an alert. For more about webhooks be sure to checkout [this post on VMware's blog][2].  There's also a great series of articles that goes into more depth about [using webhooks to automate remediation][3] and the "self-healing datacenter".

I've been working on developing a shim for Microsoft Teams because there wasn't one already and I wanted to show a couple of samples here.  Here's an example of an alert posted from Log Insight.

![Log Insight Alert](/assets/images/msteams-li-latency.png)

And here's an example of an alert from vROps.  This is a custom alert that monitors Windows services using the Endpoint Operations adapter.

![vROps Alert](/assets/images/msteams-vrops.png)

I feel like the vROps information can be improved, but I didn't want to go crazy on my initial attempt at this.  I'd like to add some fields, such as the alert status and maybe a link back to the alert in vROps.

I think that using collabloration tools for key alerts like this is an interesting way to experiement with new ways of handling IT operations.  It could also help reduce email and unclutter that inbox!  Instead of having potentially multiple email threads floating around everyone on the team can engage in the problem discussion in real time and in the appropriate Teams channel where the alert has been posted.  I'm interested in feedback or additional thoughts.  Until I get around to adding comments to this blog please feel free to reach out to me via email, Twitter, or LinkedIn.


[1]: https://github.com/vmw-loginsight/webhook-shims
[2]: https://blogs.vmware.com/management/2017/02/self-healing-datacenter.html
[3]: https://blogs.vmware.com/management/2017/01/vrealize-webhooks-infinite-integrations.html