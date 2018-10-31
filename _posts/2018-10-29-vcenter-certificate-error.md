---
layout: single
title:  "vSphere Web Client Error After Replacing Certificates"
categories: vsphere
tags: vcenter certificates
---

Managing certificates in a large vSphere environment has never been particularly fun.  Admittedly, it has improved vastly since the release of 6.x and the integrated certificate authority, but it can still be a chore to update a large environment.  I recently had to update the PSC, vCenter, and ESXi host certificates due to a looming expiration date on the CA certificate and ran into a strange issue that ultimately VMware Support helped me to track down.  I figured it was worth sharing the problem for future reference.  The TL;DR version of this story is that having too many old trusted root certificates in the VCSA's trusted root store may cause this issue.

In this environment the VMCA acts as a subordinate or intermediate CA to an internal Microsoft Certificate Services infrastructure.  There is a great [VMware blog post][5] that walks through some different approaches to managing certificates with the VMCA.  For what it's worth, the next time around I'm going to try the hybrid approach that is outlined there.

Initally, things were going pretty well.  I issued a new subordinate CA certificate for the VMCA and updated it on the external PSC. I then issued a new machine certificate for vCenter and issued new solution certificates.  All of this completed with no issues, as far as I could tell. For reference, here are the VMware docs covering both of these steps.

[Creating a Microsoft Certificate Authority Template for SSL certificate creation in vSphere 6.x][2]

[Replace VMCA Root Certificate with Custom Signing Certificate and Replace All Certificates][3]

After the vCenter services restarted I tried to access the vSphere Web Client when I was presented with the following error:

```
**A server error occurred.**
[400] An error occurred while sending an authentication request to the vCenter Single Sign-On server - An error occurred when processing the metadata during vCenter Single Sign-On setup - javax.net.ssl.SSLException: java.lang.RuntimeException: Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty.
*Check the vSphere Web Client server logs for details.*
```

I doubled checked all of the certificates.  The machine certificate and solutions certificates all had a new expiration date, showed the full CA chain correctly, and had no issues... The PSC has a handy certificate management UI where you can easily view the certificates for both the PSC and any vCenters.

![Certificate Management](/assets/images/psc-certmgmt.png)

![Trusted Root Certificates](/assets/images/psc-trustedroots.png)

The certificate manager script also has an option to revert to the previous certificates.  I tried this and although I confirmed that it restored the previous certificates successfully I was still getting the web client error.  After a healthy amount of searching and combing logs I opened a support ticket with VMware.  As usual the VMware support tech was very helpful and after checking my certs and consulting with his colleagues he wanted to try deleting some of the trusted root certificates that were no longer needed.  This has to be done using a LDAP client/explorer and JXplorer is recommended.  This KB article, though for a differnt issue, walks you through using JXplorer. [How to use JXplorer to update the LDAP string][1]  In JXplorer browse to Configuration, Certificate-Authorities and view the details for each of the certificate IDs listed.  Using the DNs shown here you can identify which certificates to delete. Make sure you take a snapshot of the PSC and linked PSCs before deleting anything.  After deleting the uneeded certificates you'll probably need to restart the PSC and vCenter.  If you prefer to just restart the services, these commands can be run in the shell:

```
service-control --stop --all
service-control --start --all
service-control --status --all
```

With the PSC and vCenter certificates renewed and the web client functioning it was time to update the hosts.  Renewing a host certificate immediately after replacing the vCenter certificate results in:
```
A general system error occurred: Unable to get signed certificate for host: esxi_hostname. Error: Start Time Error (70034)
```
This error is easy to fix, just modify the vCenter advanced setting 'vpxd.certmgmt.certs.minutesBefore' to '10'.  This is documented here: ["Signed certificate could not be retrieved due to a start time error" when adding ESXi host to vCenter Server 6.0][4]

Many host certificate renewals later my adventures in certificate management had finally ended.  Whew!  Hopefully this information helps someone, possibly even my future self.

[1]: https://kb.vmware.com/kb/2077170
[2]: https://kb.vmware.com/kb/2112009
[3]: https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.psc.doc/GUID-7F63F6D3-67E5-4C8B-B5EF-5C67F71E82B4.html
[4]: https://kb.vmware.com/kb/2123386
[5]: https://blogs.vmware.com/vsphere/2017/01/walkthrough-hybrid-ssl-certificate-replacement.html
