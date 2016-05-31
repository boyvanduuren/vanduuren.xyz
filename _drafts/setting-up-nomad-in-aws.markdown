---
title:  "A nomad cluster between two AWS EC2 regions"
date:   2016-05-30 21:04:50+02:00
description: Setting up a small nomad cluster
---

In this article we'll set up a small Nomad cluster spanning two different Amazon
EC2 regions.

# prerequisites

There will be a few things we need before we can start:

* an [AWS account][1] (free tier will suffice)
* [aws-cli installed][2]
* [aws-cli configured][3]
* some free time

When all this is done, we're ready to start.

# configuring our EC2 instances

What we'll be doing is following the [nomad getting started][4] guide with a few changes that
will lead to a cluster running two nomad servers and four nomad clients spanning
two EC2 regions.
This requires some (minimal) setup. We'll need to create [security groups][5] and [elastic IP addresses][6] so the two
instances can reach each other, and then create the instances.
After all that has been setup we'll configure the instances.

## choosing your regions

The two regions I'll be using are `eu-west-1`, which is located in Dublin, Ireland and `eu-central-1`,
which is in Frankfurt, Germany.
You can choose your own regions. Please be aware that by default you can only use the regions
US East (Northern Virginia), and US West (Oregon). If you want to be able to use different regions,
you'll have to send an email to [Amazon Web Services](mailto:aws-verification@amazon.com).

The choice you make doesn't matter for the most part. The only difference will be different
ID's for the [Amazon Machine Images][7] we'll be using to run on our instances.


## creating the security groups




[1]:    https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html
[2]:    https://docs.aws.amazon.com/cli/latest/userguide/installing.html
[3]:    https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
[4]:    https://www.nomadproject.io/intro/getting-started/install.html
[5]:    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html
[6]:    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html
[7]:    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html
