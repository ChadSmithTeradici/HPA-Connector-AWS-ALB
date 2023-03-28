# HPA-Connector-AWS-ALB
---
title: AWS ALB for HP Anyware connectors
description: Configure a AWS ALB for distrubuting PCoIP connections between multiple HP Anyware connectors
author: chad-m-smith
tags: HPA, AWS, ALB, CAC
date_published: 2023-03-28
---

---
title: Install Teradici PCoIP on AWS  EC2 Mac instance
description: Learn how to deploy PCoIP Ultra on AWS EC2 Mac instance with a Teradici subscription plan.
author: chad-m-smith
tags: Teradici, AWS, Mac, EC2
date_published: 2021-10-20
---

Chad Smith | HP Anyware Alliance Architect at HP

<p style="background-color:#CAFACA;"><i>Contributed by HP employees.</i></p>

This guide shows you how to use AWS cloud formation script to deploy a (layer 7) Application Load Balancer (ALB) to distribute PCoIP connection request between multiple HP Anyware connectors. In this architecture the connectors reside in a DMZ subnet and have elastic IP's associated to them. In coming HTTPS requests are directed to the ALB, which then uses a round robin LB policy to distribute the connection requests. After authentication and PCoIP pixel traffic is generated from the host, the pixel traffic uses the external IP address of the connector to send pixel data to the client. The load balancers ‘sticky session’ policy ensure returning requests will be directed back to the originally assigned connector. *(see diagram below)*



EC2 Mac instances are available for purchase as Dedicated Hosts through On Demand and Savings Plans pricing models. Billing for EC2 Mac instances is per second with a 24-hour minimum allocation period to comply with the Apple macOS Software License Agreement. Through On Demand, you can launch an EC2 Mac host and be up and running within minutes. At the end of the 24-hour minimum allocation period, the host can be released at any time without further commitment. 

More Information on EC2 MAC Instance can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-mac-instances.html).

## Objectives

+ Allocate a AWS EC2 Mac Instance from AWS Console.
+ Configure Security Groups to allows access to instance (SSH,VNC & PCoIP ports).
+ Install supporting software and configure security parameters within Mac OS.
+ Connect to EC2 Mac Instance via PCoIP client

## Costs

This guide uses billable components of AWS Cloud and assumes Teradici subscription, including the following:

+   [Teradici PCoIP](https://connect.teradici.com/contact-us), Teradici PCoIP subscriptions
+   [AWS EC2 Mac Instance](https://aws.amazon.com/ec2/instance-types/mac/), including vCPUs, memory, disk, and GPUs as a dedicated host.
+   [Internet egress and transfer costs](https://aws.amazon.com/blogs/architecture/overview-of-data-transfer-costs-for-common-architectures/), for PCoIP and other applications communications

Use the [AWS pricing calculator](https://calculator.aws/#/) to generate a cost estimate based on your projected usage.

## Before you begin

In this section, you set up some basic resources that the tutorial depends on.

1. Instructions in this guide assume that you have a [AWS account](https://aws.amazon.com/free/) 

1. Ensure you have [Service Quotas](https://console.aws.amazon.com/servicequotas) for **'Running Dedicated mac1 Hosts'**.

1. Familiarize yourself with [AWS network](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Networking.html) topology and best practices.

1. Obtain a [Teradici PCoIP registation](https://connect.teradici.com/contact-us) code that has at least one un-assigned seat available.

1. Have a [Teradici registerd login](https://help.teradici.com/s/login/SelfRegister) credentials in order to obtain **Graphics Agent for macOS**. 

## Set up the virtual workstation

In this section, you create and configure a virtual workstation, including setting up networking and installing utilities. 
