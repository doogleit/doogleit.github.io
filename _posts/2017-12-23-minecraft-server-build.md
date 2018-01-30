---
layout: single
title:  "Minecraft Server Build"
date:   2017-12-23 18:22:43 -0500
categories: minecraft gaming
---
I've been thinking about setting up a small Minecraft server for a while now.  It's a great game for kids and can also serve as an educational tool.  As a quick side note, if you know a child that might be interested in computer science check out the Hour of Code activities at [code.org][code] and [makecode.com][makecode].  Both have excellent resources and educational tools, some of which are related to Minecraft.  For building your own server, there are a lot of options out there and a ton of info already written on this, but I thought it might be useful to summarize my thoughts/experience.

I started experimenting with this by just running a sever locally on my home computer.  Downloading and running the 'vanilla' Minecraft server jar is pretty simple and there's a good tutorial on the [Minecraft Wiki][minecraft-wiki].  Most servers however use a number of plugins for customization and added functionality.  'Spigot' is the most popular custom server, optimized for performance, and likely the best option in most cases. There's good summary of all this custom stuff on the [Spigot Wiki][spigot-wiki].

I looked at several cloud services to host my server and eventually settled on AWS, mostly because it allows you to run a free instance for the first year.  This is a great way to start playing with building your own server in the cloud and I didn't find any other cloud providers with this long of a trial period.  If you haven't already tried AWS, new accounts have a free tier of their Elastic Compute Cloud (EC2) service for the first 12 months.  This lets you run a Linux t2.micro instance for a whole year at no charge.  The specs aren't great (1GB of memory), but it should suffice for a small server with just a few users.  This is the route I took and I haven't had any issues so far.  Find more info here: [AWS Free Tier][aws-free-tier].

I had already built a Spigot Server locally, so I just had to use SCP to copy the folder to my AWS instance.  When creating the EC2 instance I selected the Amazon Linux AMI.  I had to update the JRE to 1.8 for Spigot to run:

{% highlight shell %}
sudo yum install -y java-1.8.0-openjdk.x86_64
sudo /usr/sbin/alternatives --set java /usr/lib/jvm/jre-1.8.0-openjdk.x86_64/bin/java
sudo /usr/sbin/alternatives --set javac /usr/lib/jvm/jre-1.8.0-openjdk.x86_64/bin/javac
{% endhighlight %}

That's it! After the first year I'll probably re-evaluate where this server is going to be hosted, but for now I've found this to be a great way to start playing with the idea of hosting your own Minecraft server in the cloud.

[minecraft-wiki]: https://minecraft.gamepedia.com/Tutorials/Setting_up_a_server
[spigot-wiki]: https://www.spigotmc.org/wiki/what-is-spigot-craftbukkit-bukkit-vanilla-forg/
[aws-free-tier]: https://aws.amazon.com/free/
[build-tools]: https://www.spigotmc.org/wiki/buildtools/
[code]: https://code.org/learn
[makecode]: https://makecode.com
