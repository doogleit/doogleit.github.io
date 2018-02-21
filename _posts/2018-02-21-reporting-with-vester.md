---
layout: single
title:  "vSphere Reporting with Vester"
date:   2018-02-21 19:16:08 -0500
categories: vsphere powercli automation
---
If you're not familiar with Vester it's a really cool, light-weigth approach to configuration management for your vSphere environment.  You can check out the project on [GitHub][1] and [Brian Bunke][2] has written an excellent [introductory blog series][3] that you should absolutely check out.  One of the inital challenges I had when I first started using Vester was figuring out how I wanted to report on the output.  The console output is great for testing, but I found myself thinking about how I was going to produce a report that my fellow IT admins and/or a manager would appreciate.

Vester uses the [Pester][4] testing framework so the results can easily be exported to a XML file in NUnit format.  This allows you to integrate Vester with a Continuous Intergration (CI) system, but for time being I wanted a simpler way to convert the XML into reports.  I'd like to eventually set this up with GitLab and might have a future post on that.

[ReportUnit][5] is one of the best options I found that offers a simple way to generate nice looking HTML reports from NUnit XML files.  Their download link is broken so I initially downloaded the source from GitHub and built it in Visual Studio, but I've since found on their issues page that you can download and extract the executable from [NuGet][6]. I'd recommend downloading the [1.5 beta package][7].  The 1.2 version was not displaying the status messages in the reports when I tested it.  Once you've downloaded the NuGet package you can unzip it and copy "ReportUnit.exe" from the "tools" folder to wherever you will be generating the reports.  In this example I'm goint to use C:\reports.

Run Vester with the `-XMLOutputFile` parameter to save the results to XML.
```
Invoke-Vester -XMLOutputFile "C:\reports\dev-results.xml"
```

Then run ReportUnit.exe and provide it the XML file.
```
ReportUnit.exe "C:\reports\dev-results.xml"
```

This creates a HTML file with the same name, dev-results.html, that will look something like this:
![ReportUnit](/assets/images/reportunit-vester.png)

What's also cool is you can run ReportUnit against a whole folder of XML files and it will produce reports for all of them with an Index.html file summarizing the reports.  This is handy if you have multiple vCenters or multiple Vester config files and want to report on all of your XML output files together.  Just give ReportUnit the input folder containing your XML files and an output folder for the HTML files it creates:

```
ReportUnit.exe "input-folder" "output-folder"
```

Here's a sample of the Index it creates with links to each of the reports:
![ReportUnit](/assets/images/reportunit-index.png)

Besides being written entirely in Powershell and PowerCLI, the two things I really like about Vester are that all of your configuration is defined in text based JSON files and you can test/remediate configuration on all of your vSphere objects (clusters, datastores, datastore clusters, VDS's, hosts, and VMs).

[1]: https://github.com/WahlNetwork/Vester
[2]: http://www.brianbunke.com/
[3]: http://www.brianbunke.com/blog/2017/03/07/introducing-vester/
[4]: https://github.com/pester/Pester
[5]: https://github.com/reportunit/reportunit
[6]: https://www.nuget.org/
[7]: https://www.nuget.org/packages/ReportUnit/1.5.0-beta1


