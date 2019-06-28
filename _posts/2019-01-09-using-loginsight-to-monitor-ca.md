---
layout: single
title:  "Using Log Insight to Monitor a Windows CA"
categories: vrealize
tags: monitoring loginsight
---

Log Insight is a really versatile tool when it comes to monitoring and searching logs.  I often forget that it can be useful for a lot more than just your VMware infrastructure.  In this post I'm going to configure a Log Insight agent on a Windows Certificate Authority to send event logs to Log Insight and create an alert for specific event IDs in the Application event log.

We'll start by installing the Log Insight agent on the Windows CA. The agents can be downloaded from the Log Insight server.  Go to the top right menu and select *Administration*.  Then in the left hand navigation select *Agents* and scroll down the page if needed to *"Download Log Insight Agent"*.  Select *"Windows MSI (32-bit/64-bit)"*.  Install the MSI package on the Windows CA, accepting all the defaults.  It should already have the address of the Log Insight server filled in during the installation.

Next we'll configure the agent.  The Log Insight Marketplace has all kinds of content packs for various solutions.  First we'll add the *"Microsoft - Windows"* content pack.  Navigate to the Log Insight Menu (top right)->*Content Packs->Marketplace*, find *"Microsoft - Windows"* and install the content pack.  This will add content for analyzing Windows servers along with a configuration template for agents to forward  Windows event logs.  While we're here, go ahead and install the *"Microsoft - IIS"* content pack as well if you are using Web Enrollment and also want to get the IIS logs.

With the Windows content pack installed we're now going to configure the agent.  Go back to *Administration->Agents* and find *"Microsoft - Windows"* in the drop down.  Click the *"Copy Template"* icon on the right.

![Log Insight](/assets/images/loginsight-agent-template.png)

Provide a name for the group of agents that will receive this configuration and click *Copy*.

![Log Insight](/assets/images/loginsight-agent-group.png)

Lastly, configure the filter to include the agents that will receive this configuration. Add the Hostname using the fully qualified domain name or use the IP Address if you prefer, and save the configuration.

![Log Insight](/assets/images/loginsight-agent-filter.png)

On the Windows server you can find the agent configuration in "C:\ProgramData\VMware\Log Insight Agent".  The "liagent-effective.ini" file should now be updated with the configuration from the server to include the Windows event logs. Any new events should show up now in Log Insight.

To monitor our CA we're going to create an alert for a couple of specific events - Event ID 53 and 22.  There is a good Microsoft blog post that mentions them [here][1].  Event ID 53 is a warning event that occurs when a request is denied and Event ID 22 is an error event that occurs when a request fails.  In Log Insight under *Interactive Analytics* create a query for these two event IDs that looks like this:

![Log Insight](/assets/images/loginsight-ca-query.png)

We are looking for events from the Application event log, where the source is CertificationAuthority, and the event ID is 53 or 22.  With the query created, use the alarm bell icon on the right to "Create Alert from Query...".

![Log Insight](/assets/images/loginsight-ca-query-alert.png)

In the New Alert configuration provide a Name, Description, a Recommendation if you like, and an Email address to send the alerts to.  Under the alert conditions I don't necessarily want to receive an email every time this event is logged, so I've selected the last option to get the aggregated events over a period of time.  Set the time period to whatever you feel comfortable with, depending on how active the CA is, and click Save. 

![Log Insight](/assets/images/loginsight-ca-newalert.png)

Now we should get an email alert whenever these events occur and can act accordingly.  Here's another example to query for all error events from two providers, both the CA and the Online Reponder.

![Log Insight](/assets/images/loginsight-ca-query-all.png)

 We could include more warning events from the CA or expand it to monitor the IIS logs for web enrollment.  I'm thinking about expanding on this and might have a follow up post in the near future.  In the mean time I'd welcome any feedback or suggestions.  Find me on Twitter [@virtually_doug][2].


[1]: https://blogs.technet.microsoft.com/askds/2010/08/31/the-case-of-the-enormous-ca-database/
[2]: https://twitter.com/virtually_doug
