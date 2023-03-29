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

### Setting parameters and launching the CFT

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/CFT-Answers.png)

1. ${\color{orange}Stack \space Name}$ : This field is the name you are going to name your cloud formation deployment, name should be recognizable for function. 

2.	${\color{orange}EnvironmentName}$ : This field allows to to apply a prefix to all resources deployed by the cloud formation template. 

3.	${\color{orange}VPC}$ : The virtual private cloud that the ELB will be deployed in. All HP Anyware connector instances, subnets and security groups must all reside in this VPC.

4.	${\color{orange}Subnets}$ : This drop-list is pre-populated with all the subnets in the AWS accounts region. The ELB configuration requires that at least **two subnet public subnet in two different AZs within the same VPC** be selected, even if you plan on only utilizing a single subnet in a AZ. Your subnets should also allow for **public elastic IPs** be assigned to the HP Anyware Connectors, in order to properly route PCoIP traffic. 

5. ${\color{orange}SecurityGroup}$ : There is a limitation in this cloud formation template that only allows one security group to be applied on ELB provisioning. Depending on your organizations security policy, you may have to go back into the load balancer configuration setting and add other security groups *(shown further below)*. The security group must support all PCoIP traffic from client request to host residing in private subnets. Refer to the [Anyware Manager as a Service](https://www.teradici.com/web-help/cas_manager_as_a_service/reference/firewall_load_balancing/) administration guide for further information.

6. ${\color{orange}SSLCertificateARN}$ : Because session negotiation happens over HTTPS, the ELB configuration requires a  3rd party SSL certificate to validate the load balancer to client connection requests. To reference a SSL certificate, one must import an existing or acquired a certifcate in the [AWS Certificate Manager (ACM)](https://console.aws.amazon.com/acm). The cloud formation deployment will need the **ARN** of the saved SSL certificate. The SSLs ARN needs to be copied and pasted into the this field. To find the ARN, click on the certificate's URL and view the ARN in the certificate status page. *(see below)*

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/FIND-ARN.png)

7. ${\color{orange}LoadBalancerTopology}$ : The cloud formation template does provide the option to configure the ELB as either an internal (or) internet facing deployment. The majority of situations, an internet facing load balancer is typically deployed in cloud and is the default option.

To Launch the CFT - Press the **Create Stack** button to continue.

**Note:** the following menus in the cloud formation deployment are optional and are organization dependent.

While the cloud formation deployment runs, it will build three services the **LoadBalancer, DefaultTargetGroup** and **LoadBalancerListener**, When the entire cloudformation process completes the name of the deployment shows complete. 

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/CFT-Running.png)

Finally, the output tab is populated with the URLs of the three provisioned services, the **LoadBalancerUrl** is the DNS name that is handed out to PCoIP client needing to connect to the load balancer front-end. 

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/CFT-Output1.png?raw=true)

### Manually adding additional security groups to load balancer

As previously stated, the cloudformation template only allows one to add a single security group in a deployment. In most cases you must add additional SGs to ensure proper communication between instances and subnets. **Load Balancers >** *Prefix Name* **> Security **

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/ADD-SG.png)

### Installing and configuring the HP Anyware connector instances

### Register targets Instances and ensure healthy status
Enter in the Target Groups section of the Load Balancing menu on the left-hand side of the [EC2 dashboard](https://console.aws.amazon.com/ec2)

  ![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/ChooseTargetGroup.png)

Select the target group with the *prefix name* that was applied in the cloudformation deployment. Select the **Register targets** in the middle of the page.

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/RegisterTargets.png?raw=true)

'Click' the blue check-box next to each of the HP Anyware connectors instances that were previously configured. Once selected, Press **Include as pending below**,

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/IncludeASPending.png?raw=true)

In the review target section of the page, ensure you have selected the correct EC2 instances then press the **Register pending targets**.

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/ReviewTargets.png?raw=true)

In the target groups page, you will see the newly added instances automatically go through a health status check and show a healthy number of participants in the group.  These HP Anyware connectors are now ready to receive HTTPS requests from PCoIP clients.

![image](https://github.com/ChadSmithTeradici/HPA-Connector-AWS-ALB/blob/main/images/HealthlyStatus.png?raw=true)







