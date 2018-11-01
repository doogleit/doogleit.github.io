---
layout: single
title:  "Migrating to vRealize Orchestrator 7.5"
categories: vrealize
tags: vro automation
---

vRealize Orchestrator 7.5 does not have an upgrade path from prior versions, but there is a migration process to migrate from an older version to a new deployment.  A quick overview of the migration process, from the [VMware Docs][1]:

* Deploy and configure your new vRealize Orchestrator 7.5 environment.
* Stop the source Orchestrator services.
* Enable SSH access for each node in the source and target environments.
* Ensure that the source Orchestrator database is accessible from the target Orchestrator environment.
* Back up the source Orchestrator database, including the database schema.
Complete all workflows in the source Orchestrator environment that are in the running or waiting for input states. Workflows in these states are market as failed after migration to the target environment.

Once the vRO 7.5 OVA is deployed, SSH can be enabled in the VAMI
![vRO 7.5 Enable SSH](/assets/images/vro75-enable-ssh.png)

Or in the console by running these commands:
{% highlight shell %}
vro1:~ # chkconfig sshd
vro1:~ # service sshd start
{% endhighlight %}

The service on the source Orchestrator can be stopped by running:
{% highlight shell %}
vro1:~ # service vco-server stop
{% endhighlight %}

I was migrating from the vRO 7.3 appliance with the embedded PostgreSQL database.  By default the database cannot be accessed over the network.  I didn't see any details on how to make the source Orchestrator database available in this scenario and at first I didn't realize that this was the case.  The initial migration check failed to validate the source vRealize Orchestrator database:

![vRO 7.5 Migration Error](/assets/images/vro75-migrate-error.png)

Using netstat I confirmed that the database was only listening on the localhost IP address, 127.0.0.0:

{% highlight shell %}
vro1:~ # netstat -an | grep 5432
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:5432          127.0.0.1:44311         ESTABLISHED
tcp      111      0 127.0.0.1:60251         127.0.0.1:5432          CLOSE_WAIT
tcp      111      0 127.0.0.1:60249         127.0.0.1:5432          CLOSE_WAIT
tcp        0      0 127.0.0.1:44311         127.0.0.1:5432          ESTABLISHED
tcp      111      0 127.0.0.1:60250         127.0.0.1:5432          CLOSE_WAIT
{% endhighlight %}

After searching for information on PostgreSQL configuration, I later found that this is actually partly addressed later in the docs under [Troubleshooting the Orchestrator Migration][2].

I added the line: 
```
listen_addresses = '*' 
```
to the file `/var/vmware/vpostgres/current/pgdata/postgresql.conf` and restarted the `vpostgres` service:
{% highlight shell %}
vro1:~ # service vpostgres restart
{% endhighlight %}

With this done, netstat confirmed that it was now listening on all IP addresses.
{% highlight shell %}
vro1:~ # netstat -an | grep 5432
tcp        0      0 0.0.0.0:5432            0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:5432          127.0.0.1:44312         ESTABLISHED
tcp        0      0 :::5432                 :::*                    LISTEN
tcp      111      0 127.0.0.1:60251         127.0.0.1:5432          CLOSE_WAIT
tcp      111      0 127.0.0.1:60249         127.0.0.1:5432          CLOSE_WAIT
tcp      111      0 127.0.0.1:60250         127.0.0.1:5432          CLOSE_WAIT
tcp        0      0 127.0.0.1:44312         127.0.0.1:5432          ESTABLISHED
{% endhighlight %}

Interestingly, after this is done the migration check will give you a slightly more detailed error message with instructions to add a line to `/var/vmware/vpostgres/current/pgdata/pg_hba.conf`:
```
host  all  all  <IP address>/32  md5
```
![vRO 7.5 Migration Error](/assets/images/vro75-migrate-error2.png)

After editing the second file restart the `vpostgres` service again and with all the PostgreSQL configuration done for remote access the migration check looks good.

![vRO 7.5 Migration Check](/assets/images/vro75-migrate-check.png)

It runs through the migration process.
![vRO 7.5 Migration Process](/assets/images/vro75-migrate-process.png)

I also watched the migration log on the new appliance.
{% highlight shell %}
vro1:~ # tail -f /var/log/vco/vro-mseq.migration.log

[2018-09-28 18:37:14.687847] [740f0ade2e4740c380bdfcdcd64e547d:Running] Stop the vRealize Orchestrator services on local node.
[2018-09-28 18:37:14.691464] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M04-stop-services
[2018-09-28 18:37:23.121463] Script invocation completed with code 0
[2018-09-28 18:37:23.121551] [740f0ade2e4740c380bdfcdcd64e547d:Completed] Stop the vRealize Orchestrator services on local node.
[2018-09-28 18:37:23.126899] [275674eff5d94fe49fe20cdfcc9eef25:Running] Migrate the source vRealize Orchestrator configuration
[2018-09-28 18:37:23.129914] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M10-migrate-vro-configuration
[2018-09-28 18:37:39.441878] Script invocation completed with code 0
[2018-09-28 18:37:39.441969] [275674eff5d94fe49fe20cdfcc9eef25:Completed] Migrate the source vRealize Orchestrator configuration
[2018-09-28 18:37:39.445931] [3ad2b2a86ae54598a52a9d6950e84f54:Running] Migrate the source vRealize Orchestrator database
[2018-09-28 18:37:39.449299] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M15-migrate-vro-db
[2018-09-28 18:37:49.896334] Script invocation completed with code 0
[2018-09-28 18:37:49.896442] [3ad2b2a86ae54598a52a9d6950e84f54:Completed] Migrate the source vRealize Orchestrator database
[2018-09-28 18:37:49.901129] [8656b017c36d4a7b82c1480fdeab5544:Running] Clean up the migrated vRealize Orchestrator database.
[2018-09-28 18:37:49.904478] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M20-cleanup-db
[2018-09-28 18:37:53.838100] Script invocation completed with code 0
[2018-09-28 18:37:53.838203] [8656b017c36d4a7b82c1480fdeab5544:Completed] Clean up the migrated vRealize Orchestrator database.
[2018-09-28 18:37:53.842301] [f43b252676494071bc4f52bc5151d395:Running] Reinstall the vRealize Orchestrator plug-ins on local node.
[2018-09-28 18:37:53.845902] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M21-reinstall-plugins
[2018-09-28 18:37:54.001940] Script invocation completed with code 0
[2018-09-28 18:37:54.002049] [f43b252676494071bc4f52bc5151d395:Completed] Reinstall the vRealize Orchestrator plug-ins on local node.
[2018-09-28 18:37:54.006549] [58d0c2f4637f48ada31c2e8908e32110:Running] Start the vRealize Orchestrator server on local node.
[2018-09-28 18:37:54.009551] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M30-start-vco-server
[2018-09-28 18:39:05.712970] Script invocation completed with code 0
[2018-09-28 18:39:05.713081] [58d0c2f4637f48ada31c2e8908e32110:Completed] Start the vRealize Orchestrator server on local node.
[2018-09-28 18:39:05.718210] [a5526b1a5be04b34bb8b846f5520ada1:Running] Start the vRealize Orchestrator Control Center on local node.
[2018-09-28 18:39:05.721643] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M45-start-vco-configurator
[2018-09-28 18:39:27.766123] Script invocation completed with code 0
[2018-09-28 18:39:27.766232] [a5526b1a5be04b34bb8b846f5520ada1:Completed] Start the vRealize Orchestrator Control Center on local node.
[2018-09-28 18:39:27.770520] [cb0d214b21414994b9ee777815fcc9d5:Running] Finalize migration.
[2018-09-28 18:39:27.773423] Invoke script /usr/lib/vcac/tools/vro-migration/sequence/migration/scripts/M99-cleanup
[2018-09-28 18:39:31.641656] Script invocation completed with code 0
[2018-09-28 18:39:31.641754] [cb0d214b21414994b9ee777815fcc9d5:Completed] Finalize migration.
[2018-09-28 18:39:31.646656] Sequence execution finished
[2018-09-28 18:39:31.650003] Sequence state changed to [migration.completed]
[2018-09-28 18:39:31.650077] Remove /storage/vro-migrate/target_vro_rootpass
[2018-09-28 18:39:31.650145] Remove /storage/vro-migrate/source_vro_rootpass
[2018-09-28 18:39:31.650202] Remove /storage/vro-migrate
[2018-09-28 18:39:31.650312] Uninstall sshpass command
[2018-09-28 18:39:31.661025] Sequence finalize
{% endhighlight %}

With the migration complete it gives you some post migration steps, including a link to the [VMware Docs][3] with more information on the post migration.  
![vRO 7.5 Migration Complete](/assets/images/vro75-migrate-complete.png)

One thing worth noting is that LDAP authentication is no longer available.  If the old/source Orchestrator was using LDAP for authentication that will need to be re-configured to use vSphere or vRA.

[1]: https://docs.vmware.com/en/vRealize-Orchestrator/7.5/com.vmware.vrealize.orchestrator-migrate.doc/GUID-F57EB2FF-F0BF-4471-80FC-BBC725836F1E.html
[2]: https://docs.vmware.com/en/vRealize-Orchestrator/7.5/com.vmware.vrealize.orchestrator-migrate.doc/GUID-A7446313-5319-4E34-B123-BE701F572DF4.html
[3]: https://docs.vmware.com/en/vRealize-Orchestrator/7.5/com.vmware.vrealize.orchestrator-migrate.doc/GUID-A419F87E-5C4E-4726-B69E-081F2B485F07.html
