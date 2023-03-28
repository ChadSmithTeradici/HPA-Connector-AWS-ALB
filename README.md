---
title: AWS ALB for HP Anyware connectors
description: Configure a AWS ALB for distrubuting PCoIP connections between multiple HP Anyware connectors
author: chad-m-smith
tags: HPA, AWS, ELB, ALB, CAC, PCoIP
date_published: 2023-03-28
---

${\color{green}Contributed \space by \space HP \space employee.}$

This guide shows you how to use a AWS cloud formation script (CFT) to deploy a layer 7 - Application Load Balancer (ALB) to distribute PCoIP connection request between multiple HP Anyware connectors. In this architecture, the connectors reside in a public subnet and have elastic IP's associated to them. Incoming HTTPS requests are directed to the ALB, which then uses a 'round robin' LB policy to distribute the connection requests. After authentication and PCoIP pixel traffic is generated from the PCoIP host, the pixel traffic is routed to the external IP address of the connector thats send pixel data directly to the client. The load balancers ‘sticky session’ policy ensure returning requests are directed to the assigned connector. *(see diagram below)*

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/AWS-ALB-topology.png)
 
HP Anyware connectors run on EC2 instances, its installation and configuration are outside the scope of this guide. Refer to the [Anyware Manager as a Service]( https://www.teradici.com/web-help/cas_manager_as_a_service/?_ga=2.105362883.1952229980.1680021434-1440526573.1672767407) deployment guide on ‘how to’ configure the connector instances. Once the connectors have been deployed, you can register the instances into the target group created by the load balancer script. 

More Information on AWS Elastic Load Balancing (ELB) service can be found [here](https://aws.amazon.com/elasticloadbalancing/).

## Objectives

+ Load the *HPA-Connector-AWS-ALB.yml* script in the AWS cloudformation deployment service
+ Configure the script's variables for your existing VPC topology.
+ After HPA connector(s) deployment, come back to AWS ELB target group and register instances
+ Verify Health Status of each HPA connector.

## Costs

This guide uses billable components of AWS Cloud and assumes Teradici subscription, including the following:

+   [Teradici PCoIP](https://connect.teradici.com/contact-us), Teradici PCoIP subscriptions
+   [AWS EC2 Instance types](https://aws.amazon.com/ec2/instance-types/), including vCPUs, memory and disk for instances that will run as connectors .
+   [Internet egress and transfer costs](https://aws.amazon.com/blogs/architecture/overview-of-data-transfer-costs-for-common-architectures/), for PCoIP and other applications communications
+   [AWS Elastic Load Balancing costs](https://aws.amazon.com/elasticloadbalancing/pricing/), for ELB services that CFT will provision

Use the [AWS pricing calculator](https://calculator.aws/#/) to generate a cost estimate based on your projected usage.

## Before you begin

In this section, you set up some basic resources that the tutorial depends on.

1. Instructions in this guide assume that you have a [AWS account](https://aws.amazon.com/free/) 

1. Ensure you have [Service Quotas](https://console.aws.amazon.com/servicequotas) for **'Elastic Load Balancing'**.

1. Familiarize yourself with [AWS network](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Networking.html) topology and best practices.

1. Obtain a [Teradici PCoIP registation](https://connect.teradici.com/contact-us) code that will be used for HPA manager licenses (*out of scope*).

1. Have a [Teradici registerd login](https://help.teradici.com/s/login/SelfRegister) credentials in order to open support ticket(s) if needed. 

## Set up the Elastic Load Balancer

In this section, the cloudformation wizard will ingest the **HPA-Connector-AWS-ALB.yml** which resides in a publicly accessible S3 bucket, which launching the cloudformation deployment service. Answer the minimum number of configuration questions based on your environment’s topology. Finally, address some caveats and ‘gotchas’ that could cause issues with the ELB deployment.

**'CTRL + CLICK'** On the AWS cloudformation graphic to open a new browser tab to start the cloudformation deployment wizard:

[![name](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/CFT-Button.png?raw=true)](https://console.aws.amazon.com/cloudformation/home?#/stacks/quickcreate?templateURL=https://hpagpicbucket4scripts.s3.us-west-2.amazonaws.com/HPA-Connector-AWS-ALB.yml)

## Setting parameters and launching the CFT

![image](https://github.com/ChadSmithTeradici/PCoIP-Power-Tools-via-CFT/blob/main/images/CFT-Questions.png)

1.	**VcpId** : This field is the virtual private cloud within your AWS region in which you want to deploy the solution in. If no other VPC has been created, you can the *(Default)* VPC automatically created.
2.	**SubnetId:** This field you MUST select a subnet that has been generated in the same VPC associated above. If no other subnet has been created, you can use the *(Default)* subnet automatically.
3.	**Instance Type:** This drop-list is pre-populated with NVIDIA enabled instance types. Care must be taken to ensure that your selected instances IS available in your AWS region and Available Zone (AZ). As a general rule, almost all G4dn instance types are available in all regions, which G5 instances are available in most popular regions. Check these webpages to verify your instance resource requirements. [AWS G4dn instance family](https://aws.amazon.com/ec2/instance-types/g4/)  and [AWS G5 instance family](https://aws.amazon.com/ec2/instance-types/g5/)
4.	**KeyName:** Select the Key name of your Key Pair. The key pair will be used to generate your Windows Administrator password after the Instance has been started. If you haven’t generated a Key Pair, log into the [EC2 dashboard](https://console.aws.amazon.com/ec2), navigate to the left-side bar under Network & Security > Key Pairs > create new Key Pair
5.	**RDPLocation:** The automatically generated security group associated to the instance has a rule to allow a back-door RDP (port 3889) communications to the instance for troubleshooting and HP Anywhere patching. We recommend locking this port down by entering in your workstations IP address. This address can be seen [What Is My IP?](https://www.whatismyip.com/) Quickly See your IP Address. Once your external IP has been identified, you must append a */32* to create the security group.

To Launch the CFT - Press the **Create Stack** button to continue.

## Gathering Login info IP / credentials and access the EC2 Instance via HP Anyware client

In this section, you will establish a connection to your instance using PCoIP. You will need to install a PCoIP client on your client system that will be used to initiate the session to the EC2 Instance in AWS. Depending on your network topology, use will either connect to the local IP (or) ephemeral/elastic Public IP (or) Fully Qualified Domain Names (FQDN)



