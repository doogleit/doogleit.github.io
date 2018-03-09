---
layout: single
title:  "Reporting on HPE Warranty/Entitlements from OneView"
date:   2018-03-09 17:38:17 -0500
categories: automation powershell
---

Getting the warranty info or support entitlements for a long list of HPE servers can be a bit daunting using the HPE support website.  You can enter serial numbers and product numbers in batches of up to 20 at a time and hope that it finds matches for all of them.  Not too bad, but definitely not ideal for a large environment.  If you have HPE OneView configured for Remote Support it will gather all of the warranty and contract information for you and list the details for each server and enclosure in the Remote Support section of each individual device.  Surprisingly, there isn't any kind of report with this information built into OneView and the information is only found on each individual device.

Fortunately, HPE has had pretty good Powershell support for their products.  There are modules available for just about everything on either their support site or GitHub.  OneView cmdlets can be found on GitHub at [HewlettPackard/POSH-HPOneView][1] and cmdlets for iLOs, OAs, and others can be found on their [Scripting Tools for Windows PowerShell][2] site.  The OneView module is also available in Microsoft's PowerShell Gallery starting with version 3.10.

Version 3.10 of the OneView cmdlets introduced a new cmdlet called `Get-HPOVRemoteSupportEntitlementStatus`.  This sounded like just what I needed.  I found that I could get the support entitlements for all of my enclosures or servers by running something as simple as:

```
Get-HPOVServer | Get-HPOVRemoteSupportEntitlementStatus

ResourceName ResourceSerialNumber IsEntitled EntitlementPackage EntitlementStatus CoverageDays ObligationEndDate
------------ -------------------- ---------- ------------------ ----------------- ------------ -----------------
             XXXXXXXXXX           True       CriticalService    VALID             7Hol         5/18/2018 8:00:00 PM
             XXXXXXXXXX           True       CriticalService    VALID             7Hol         5/28/2018 8:00:00 PM

```

Of course the output only contains results about the support entitlements and none of the enclosure/server details except for the serial number.  I wrote a short PowerShell script, `Get-HPOVWarrantyReport.ps1`, that will get both enclosures and servers together and combine the hardware and support information into a custom PSObject that can be passed along the pipeline or exported to CSV. You can find the script in my [GitHub Repository][3].  The default output looks like this:

```
.\Get-HPOVWarrantyReport.ps1 -Appliance oneview.mydomain.local

Name                  Model                ProductNumber SerialNumber ObligationID ObligationEndDate
----                  -----                ------------- ------------ ------------ -----------------
ChassisName01, bay 1  ProLiant BL660c Gen9 728352-B21    XXXXXXXXXX   000000000000 5/18/2018 8:00:00 PM
ChassisName01, bay 2  ProLiant BL660c Gen9 728352-B21    XXXXXXXXXX   000000000000 5/28/2018 8:00:00 PM
```

If you've already connected to a OneView appliance using `Connect-HPOVMgmt` you can omit the Appliance parameter.  You can also specify an output file to save the results directly to a CSV file and forego the console output.

```
.\Get-HPOVWarrantyReport.ps1 -Output "Warranty Report.csv"
```

If you don't have OneView or Remote Support setup I stumbled across another PowerShell solution that looks like it is worth taking a look at.  There's an HPWarranty module also on GitHub here [dotps1/HPWarranty][4].

[1]: https://hewlettpackard.github.io/POSH-HPOneView
[2]: https://www.hpe.com/servers/powershell
[3]: https://github.com/doogleit/hpe-oneview-misc
[4]: https://github.com/dotps1/HPWarranty
