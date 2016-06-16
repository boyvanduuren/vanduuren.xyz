---
title:  "A nomad cluster between two AWS EC2 regions"
date:   2016-06-01 22:34:24+02:00
description: Setting up a small nomad cluster
---

In this article we'll set up a small Nomad cluster spanning two different Amazon
EC2 regions.

<font color='red'>IMPORTANT:</font>
The resulting cluster is just something fun to experiment with. Don't run
production on it. Also, you could just use [Availability Zones][az] in stead
of two different regions.

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
This requires some (minimal) setup. We'll need to create [elastic IP addresses][6] and [security groups][5]
so the two instances can reach each other, and then create the instances.
After all that has been setup we'll configure the instances.

## choosing your regions

The two regions I'll be using are `eu-west-1`, which is located in Dublin, Ireland and `eu-central-1`,
which is in Frankfurt, Germany.
You can choose your own regions. Please be aware that by default you can only use the regions
US East (Northern Virginia), and US West (Oregon). If you want to be able to use different regions,
you'll have to send an email to [Amazon Web Services](mailto:aws-verification@amazon.com).

The choice you make doesn't matter for the most part. The only difference will be different
ID's for the [Amazon Machine Images][7] we'll be using to run on our instances.

## creating the elastic IP addresses

We'll start by creating the elastic IP addresses, these are needed when we create the security groups.
The `aws ec2 allocate-address` command will be of use here. One IP will be allocated per region.

```bash
$ aws ec2 allocate-address \
    --domain vpc \
    --region eu-west-1

          {
              "PublicIp": "203.0.113.10",
              "Domain": "vpc",
              "AllocationId": "eipalloc-64d5890a"
          }

$ # we're interested in the AllocationId and
$ # the PublicIp, so we'll save it in a variable
$ ALLOCID_WEST="eipalloc-64d5890a"
$ PUBLICIP_WEST="203.0.113.10"

$ # we'll do the same for the second region
$ aws ec2 allocate-address \
    --domain vpc \
    --region eu-central-1

          {
              "PublicIp": "203.0.113.6",
              "Domain": "vpc",
              "AllocationId": "eipalloc-1a90338f"
          }

$ ALLOCID_CENTRAL="eipalloc-1a90338f"
$ PUBLICIP_CENTRAL="203.0.113.6"

```

We now have all the information for the next step...

## creating the security groups

The security groups will be very simple. SSH access will be allowed from our own IP address
so we can remotely access both machines. 
On top of that, all traffic between the two elastic IP addresses will be allowed. That way
the nomad agents can talk to each other, and we can connect to our test jobs (which will be redis containers).

The commands we'll be using are `aws ec2 create-security-group` and `aws ec2 authorize-security-group-ingress`.

```bash
$ # First save our own public IP to a variable
$ PUBLICIP_SELF=$(curl https://dynamicdns.park-your-domain.com/getip)

$ # Configure our first region
$ aws ec2 create-security-group \
    --description "Nomad development security group" \
    --group-name nomad-dev \
    --region eu-west-1

$ aws ec2 authorize-security-group-ingress \
    --group-name nomad-dev \
    --region eu-west-1 \
    --protocol tcp \
    --port 22 \
    --cidr ${PUBLICIP_SELF}/32

$ aws ec2 authorize-security-group-ingress \
    --group-name nomad-dev \
    --region eu-west-1 \
    --protocol all \
    --port 0-65535 \
    --cidr ${PUBLICIP_CENTRAL}/32
    
$ # Configure our second region
$ aws ec2 create-security-group \
    --description "Nomad development security group" \
    --group-name nomad-dev \
    --region eu-central-1

$ aws ec2 authorize-security-group-ingress \
    --group-name nomad-dev \
    --region eu-central-1 \
    --protocol tcp \
    --port 22 \
    --cidr ${PUBLICIP_SELF}/32

$ aws ec2 authorize-security-group-ingress \
    --group-name nomad-dev \
    --region eu-central-1 \
    --protocol all \
    --port 0-65535 \
    --cidr ${PUBLICIP_WEST}/32
```

## creating our authentication keys

To be able to login to our instances, we're going to need authentication keys.
One for each region.

```bash
$ aws ec2 create-key-pair \
    --key-name nomad-west-key \
    --region eu-west-1 \
    --query 'KeyMaterial' \
    --output text > ~/.aws/nomad-west-key.pem

$ aws ec2 create-key-pair \
    --key-name nomad-central-key \
    --region eu-central-1 \
    --query 'KeyMaterial' \
    --output text > ~/.aws/nomad-central-key.pem

$ chmod 0400 ~/.aws/nomad-{west,central}-key.pem
```

## creating the instances

After all this work, we're almost ready to spin up our instances!
First, we need the AMI id's of the images we want to use. I'm going to use
the Amazon Linux AMI provided by Amazon, but of course you can choose whatever
you like. As long as Nomad can run on it!

```bash
$ aws ec2 describe-images \
    --region eu-west-1 \
    --owners amazon \
    --filters "Name=name,Values=amzn-ami-hvm-2016.03.1.x86_64-gp2" \
    --query 'Images[0].ImageId'

"ami-79aa230a"

$ AMI_WEST="ami-79aa230a"

$ aws ec2 describe-images \
    --region eu-central-1 \
    --owners amazon \
    --filters "Name=name,Values=amzn-ami-hvm-2016.03.1.x86_64-gp2" \
    --query 'Images[0].ImageId'

"ami-d8c123b7"

$ AMI_CENTRAL="ami-d8c123b7"
```

And now we're finally all set.

```bash
$ aws ec2 run-instances \
    --region eu-west-1 \
    --image-id ${AMI_WEST} \
    --security-groups nomad-dev \
    --count 1 \
    --instance-type t2.micro \
    --key-name nomad-west-key \
    --query 'Instances[0].InstanceId'
    
"i-ec3e1e2k"

$ INSTANCE_WEST="i-ec3e1e2k"

$ aws ec2 run-instances \
    --region eu-central-1 \
    --image-id ${AMI_CENTRAL} \
    --security-groups nomad-dev \
    --count 1 \
    --instance-type t2.micro \
    --key-name nomad-central-key \
    --query 'Instances[0].InstanceId'
    
"i-a2fc9213"

$ INSTANCE_CENTRAL="i-a2fc9213"


$ # Note that you might have to wait a bit
$ # before executing the next two steps

$ aws ec2 associate-address \
    --region eu-west-1 \
    --instance-id ${INSTANCE_WEST} \
    --allocation-id ${ALLOCID_WEST}

$ aws ec2 associate-address \
    --region eu-central-1 \
    --instance-id ${INSTANCE_CENTRAL} \
    --allocation-id ${ALLOCID_CENTRAL}
```

And we're up! Test if you can connect to the instances.

```bash
$ ssh -i ~/.aws/nomad-west-key.pem \
    ec2-user@${PUBLICIP_WEST} whoami

ec2-user

$ ssh -i ~/.aws/nomad-central-key.pem \
    ec2-user@${PUBLICIP_central} whoami

ec2-user
```

# setting up our Nomad cluster

It's now time to finally start setting up Nomad!
As I said before, this will be a simple setup heavily inspired by
the official Getting Started guide. We won't use Consul or anything like that.

## installing the binary

We start by logging into one of our instances and then installing the Nomad binary, and docker.

```bash
$ ssh -i ~/.aws/nomad-west-key.pem \
    ec2-user@${PUBLICIP_WEST}
$ sudo yum install -y unzip docker
$ curl -O https://releases.hashicorp.com/nomad/0.3.2/nomad_0.3.2_linux_amd64.zip
$ sudo unzip nomad_0.3.2_linux_amd64.zip -d /usr/local/bin/
$ sudo chkconfig docker on
$ sudo service docker start
```

Repeat this for the other instance.


## creating the server configuration

Now place the following text in a file named `server.conf` on both instances.
Take care to use the IP addresses in the `advertise` block.

```bash
# Increase log verbosity
log_level = "INFO"

# Bind to all interfaces
bind_addr = "0.0.0.0"

# Setup data dir
data_dir = "/tmp/server"

# Change the region to the proper aws region
region = "global"
datacenter = "aws-ec2"

# Enable the server
server {
    enabled = true
    bootstrap_expect = 2
}

advertise {
    # make sure to replace the IP with the proper id
    # for this instance!
    http = "52.50.233.53:4646"
    rpc = "52.50.233.53:4647"
    serf = "52.50.233.53:4648"
}
```

Run nomad on both instances. We're going to be dirty and redirect
`STDOUT` and `STDERR` to a log file and just background the process.
Of course, normally you'd make a proper unit file for your nomad server.

```bash
$ sudo nomad agent -config server.conf &>/tmp/nomad_server.log &
```

We now have two nomad servers that don't know of each others existence.
To create the cluster, we're going to join one of the servers to the other.

```bash
$ nomad server-join 52.50.233.53
$ nomad server-members

Name                     Address       Port  Status  Leader  Protocol  Build  Datacenter  Region
ip-172-31-19-237.global  52.29.92.34   4648  alive   false   2         0.3.2  dc1         global
ip-172-31-46-199.global  52.50.233.53  4648  alive   true    2         0.3.2  dc1         global
```

## creating client configuration

We're almost there. Let's create four client configs, and start the agents.
Place the following file in `client1.conf`.

```bash
# Increase log verbosity
log_level = "DEBUG"

# Setup data dir
data_dir = "/tmp/client1"

region = "global"
datacenter = "aws-ec2"

# Enable the client
client {
    enabled = true
    servers = ["127.0.0.1:4647"]
}

# Modify our port to avoid a collision with server1
ports {
    http = 5656
}
```

Copy that file to `client2.conf` and change `data_dir = "/tmp/client2"` and `http = 5657`.
Do the same for the other instance.

We can now start the clients. On both instances, run the following commands:

```bash
$ sudo nomad agent -config client1.conf &>/tmp/nomad_client1.log&
$ sudo nomad agent -config client2.conf &>/tmp/nomad_client2.log&
```

And a cluster with some servers and clients has been born! :-)

```bash
$ nomad node-status
ID        DC       Name              Class   Drain  Status
ef11478f  aws-ec2  ip-172-31-19-237  <none>  false  ready
14ed37a9  aws-ec2  ip-172-31-19-237  <none>  false  ready
4b289230  aws-ec2  ip-172-31-46-199  <none>  false  ready
9db8d155  aws-ec2  ip-172-31-46-199  <none>  false  ready
```

## scheduling some jobs

Let's create a simple job and deploy it on all our clients.

```bash
$ nomad init

Example job file written to example.nomad

$ sed -i 's/dc1/aws-ec2/g' example.nomad
$ sed -i 's/# count = 1/count = 4/' example.nomad
$ nomad run example.nomad

==> Monitoring evaluation "4144a94f"
    Evaluation triggered by job "example"
    Allocation "74ec6eca" created: node "8bb6af48", group "cache"
    Allocation "cc59dcf9" created: node "b71a91ec", group "cache"
    Allocation "f1685c9b" created: node "92fc2c9b", group "cache"
    Allocation "f3a4ecf5" created: node "b109e2e5", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "4144a94f" finished with status "complete"
```

And we just deployed four redis instances, one on each nomad client!
Let's see if we can connect to a redis instance from one node to the other.

```bash
$ nc 52.51.141.75 32908
ping
+PONG
```

And it works! Cool!

# conclusion

Well, that's it for now. We have a small cluster on which we can schedule jobs.
Next time we'll add a Consul registry to our setup and see what kind of cool
stuff we can do with that.

[1]:    https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html
[2]:    https://docs.aws.amazon.com/cli/latest/userguide/installing.html
[3]:    https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
[4]:    https://www.nomadproject.io/intro/getting-started/install.html
[5]:    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html
[6]:    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html
[7]:    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html
[az]:   https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-available-regions
