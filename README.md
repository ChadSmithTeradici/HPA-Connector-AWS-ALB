---
title: AWS ALB for HP Anyware connectors
description: Configure a AWS ALB for distrubuting PCoIP connections between multiple HP Anyware connectors
author: chad-m-smith
tags: HPA, AWS, ALB, CAC
date_published: 2023-03-28
---

Chad Smith | HP Anyware Alliance Architect at HP

${\color{green}Contributed \space by \space HP \space employee.}$

This guide shows you how to use a AWS cloud formation script (CFT) to deploy a layer 7 - Application Load Balancer (ALB) to distribute PCoIP connection request between multiple HP Anyware connectors. In this architecture, the connectors reside in a public subnet and have elastic IP's associated to them. Incoming HTTPS requests are directed to the ALB, which then uses a 'round robin' LB policy to distribute the connection requests. After authentication and PCoIP pixel traffic is generated from the PCoIP host, the pixel traffic is routed to the external IP address of the connector thats send pixel data directly to the client. The load balancers ‘sticky session’ policy ensure returning requests are directed to the assigned connector. *(see diagram below)*

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/AWS-ALB-topology.png)
 
HP Anyware connectors run on EC2 instances, its installation and configuration are outside the scope of this guide. Refer to the [Anyware Manager as a Service]( https://www.teradici.com/web-help/cas_manager_as_a_service/?_ga=2.105362883.1952229980.1680021434-1440526573.1672767407) deployment guide on ‘how to’ configure the connector instances. Once the connectors have been deployed, you can register the instances into the target group created by the load balancer script. 

More Information on AWS MAC Instance can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-mac-instances.html).

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
+   [AWS Elastic Load Balancing costs](https://aws.amazon.com/elasticloadbalancing/pricing/)

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
