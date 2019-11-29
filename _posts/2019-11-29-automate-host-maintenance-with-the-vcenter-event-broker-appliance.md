---
title:  "Automate Host Maintenance with the vCenter Event Broker Appliance"
date: 2019-11-29 11:00:00 -0400
categories: vSphere
tags: vcenter automation veba
---

The vCenter Event Broker Appliance is a new [VMware Fling][2] and open source project that was recently [released at VMworld Europe][1].  A preview of it was also presented at VMworld US which I had the pleasure of attending.  If you haven't seen either of these sessions I highly recommend watching the recording of [#CODE1379E "If This Then That" for vSphere - The Power of Event Driven Automation session][11].  I'm quite interested in this project and really excited to see all of the cool automation the community does with it.

If you haven't deployed the appliance the [Getting Started][3] guide has all the details for getting it setup.  Also be sure to join the [VMware {Code} Slack channel][4]. All of my work is based on the existing samples [on GitHub][5] and Opvizor's excellent article: [Audit VM configuration changes using the vCenter Event Broker][6].  A huge thanks to these guys for all of their work.

This example will disable alarm actions on a host while it is in maintenance mode.  It is actually two functions using the same script.  The first function subscribes to the `entered.maintenance.mode` event to run when a host is put into maintenance mode and disable its alarms.  The second function subscribes to the `exit.maintenance.mode` event to re-enable alarms when the host exits maintenance mode.  The script looks at the event type (or in OpenFaaS speak the "topic") that called the function and performs the appropriate action to disable/enable alarms.  Here is what the stack.yml file looks like with both functions:

```yaml
version: 1.0
provider:
  name: faas
  gateway: https://veba.yourdomain.com
functions:
  powercli-entermaint:
    lang: powercli
    handler: ./handler
    image: doogleit/powercli-hostmaint:latest
    environment:
      write_debug: true
      read_debug: true
      function_debug: false
    secrets:
      - vcconfig
    annotations:
      topic: entered.maintenance.mode
  powercli-exitmaint:
    lang: powercli
    handler: ./handler
    image: doogleit/powercli-hostmaint:latest
    environment:
      write_debug: true
      read_debug: true
      function_debug: false
    secrets:
      - vcconfig
    annotations:
      topic: exit.maintenance.mode
```

## Deploying the functions
To deploy these functions clone the example from GitHub:

```shell
git clone https://github.com/doogleit/vcenter-event-broker-appliance
cd vcenter-event-broker-appliance/examples/powercli/hostmaint-alarms
```

If you've already created a secret named 'vcconfig' from one of the other PowerCLI examples and want to use the same vCenter server and credentials you can skip this next step.  Note that the 'vcconfig' secret used in the Python tagging example is different and won't work for these functions.  Configure your vCenter details in the vcconfig.json file and create the secret:

```json
{
    "VC" : "vcenter-hostname",
    "VC_USERNAME" : "veba@vsphere.local",
    "VC_PASSWORD" : "FillMeIn"
}
```
```shell
faas-cli secret create vcconfig --from-file=vcconfig.json --tls-no-verify
```

Finally modify the gateway in the stack.yml file with your vCenter Event Broker Appliance address and deploy the functions:

```yaml
provider:
  name: faas
  gateway: https://veba.yourdomain.com
...
```
```shell
faas-cli deploy -f stack.yml --tls-no-verify
```

Have a host enter and exit maintenance mode to test it out.  You can check the logs using either:

```shell
faas-cli logs powercli-entermaint --follow --tls-no-verify
```

for a host entering maintenance mode or:
```shell
faas-cli logs powercli-exitmaint --follow --tls-no-verify
```

for a host exiting maintenance mode.

If you use SolarWinds to monitor your ESXi hosts keep an eye out for an example on that as well.  I hope to have an example on GitHub soon.

I was previously using [VMware Dispatch][7] to experiment with running functions based on vCenter event subscriptions.  If your interested you can check out some of my previous articles on using Dispatch here:

* [Event Based Automation with VMware Dispatch][8]
* [Automating Host Maintenance with VMware Dispatch][9]
* [Automating Host Maintenance with VMware Dispatch Part 2][10]

[1]: https://www.virtuallyghetto.com/2019/11/vcenter-event-broker-appliance-updates-vmworld-fling-community-open-source.html
[2]: https://flings.vmware.com/vcenter-event-broker-appliance
[3]: https://github.com/vmware-samples/vcenter-event-broker-appliance/blob/master/getting-started.md
[4]: https://vmwarecode.slack.com/archives/CQLT9B5AA
[5]: https://github.com/vmware-samples/vcenter-event-broker-appliance
[6]: https://www.opvizor.com/audit-vm-configuration-changes-using-the-vcenter-event-broker
[7]: https://vmware.github.io/dispatch/
[8]: https://doogleit.github.io/2019/07/event-based-automation-with-dispatch/
[9]: https://doogleit.github.io/2019/07/automating-host-maintenance-with-dispatch/
[10]: https://doogleit.github.io/2019/08/automating-host-maintenance-with-dispatch-part-2/
[11]: https://videos.vmworld.com/global/2019/videoplayer/29523
