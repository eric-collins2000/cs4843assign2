# cs4843assign2

![image](https://user-images.githubusercontent.com/93015308/163276641-277665b9-8869-49c1-9523-7abb20d49d79.png)


# Network

High level description:

Network YAML provided is a template for the build out of the network. It creates a Class B network which will contain the entirety of the network and all items connected to it. It deploys two availability zones inside our virtual private network which mirror each other in everything except the Class C network addresses (10.0.0.0/24 and 1.0/24 for public for example) 

Past the AWS gateway our VPC is managed by an elastic load balancer. In AWS applications availability zones refer to the ability for our VPC to judge the usage of a given redundant network as a whole, and distribute users and their load appropriately.  In our example this is two networks, but it would be very easy to make this scale to our budget, as that is the restrictive measure generally accepted in practice.  AWS offers network load balancing as well as gateway and application load balancing. From here working our way in everything is duplicated, or mirrored. In our example twice, but as mentioned above this can generally be scaled to whatever our need or budget can supply. Directly inside we have an elastic IP connected to our NAT gateway. This allows us to have fail-resistance built in. Our NAT gateway separates the internal components of our network from the outside world. Everything inside here is on the private subnet. 

## Components

![image](https://user-images.githubusercontent.com/93015308/163276765-b8ef4159-bbe4-436f-b7e0-e4eb8cd2ac84.png)

The point of presence of our network is this. It is the edge of our network and where users will come from the internet as a whole to our domain.

![image](https://user-images.githubusercontent.com/93015308/163277716-88ce1bb6-52cd-42c2-b327-b22c829464dd.png)

Our Elastic Load Balancer. This is the component that automatically shifts new users to the least used availability zone.

![image](https://user-images.githubusercontent.com/93015308/163277743-2a02a87d-bf93-4aea-865b-7d6887ad5402.png)

Our VPC or Virtual Private Cloud. This is the entirety of our personal network.

![image](https://user-images.githubusercontent.com/93015308/163278345-31098fef-2dbd-45ee-b217-3ae4453968d7.png)

This is our NAT gateway, but there is quite a bit in here. The NAT gateway both assigns IPs for our private network, as well as shields it and acts as a DMZ (or Demilitarized Zone) it is also covered under our elastic IP setting. This is the forward facing front to our network, and is therefore also our public subnet. 

![image](https://user-images.githubusercontent.com/93015308/163278645-12cc5a72-8e8f-47ce-8fff-ede97a21f86f.png)

This is our actual web application. Here is the logic that runs whatever it is that we do. Everything is housed here-except the data we save. It might be helpful to think of this as a cognition of our setup, but not the long term memory. It autoscales per our setting parameters, and is the first stop in our private network. It will handle user requests but seemlessly interface with the database itself, it is also part of the only difference in our availability zones. This difference manafests as a link from any zone to the first zone's SQL database.

![image](https://user-images.githubusercontent.com/93015308/163279040-02eae5a8-e67f-43f3-8827-cf4486858dd5.png)

Here we see both zone's link to the SQL database. As is detailed in greater depth later in this README, it is the only asymetrically designed portion of our network. As all links point to the primary database, and the primary database points to the secondary which resides in the second zone.

## Parameters

![image](https://user-images.githubusercontent.com/93015308/163276952-95c4afab-1814-410e-a46a-ac11e205d3c0.png)

This will be the basic IP topology of the network. /16 gives us a Class B which is over 65 thousand individual IPs. However we will be slicing out at least 4 Class C (254) addresses for use in our networks. Not that this matters as anything behind the NAT gateways will be masked from the exterior. This may pose an interesting problem later when we consolidate the SQL database servers. However, at setup we needn't worry as we have statically assigned the networks.

![image](https://user-images.githubusercontent.com/93015308/163277520-ec081144-23aa-421e-bbe5-f4b7ae609fb9.png)

This is the VPC's attachement to the IG

![image](https://user-images.githubusercontent.com/93015308/163279920-d5aeb34a-a6d3-4e01-8248-a8cdec2c7410.png)

The setup for each subnet.

![image](https://user-images.githubusercontent.com/93015308/163280025-3ee7978d-2d3b-40b5-b843-e9c947f4aa32.png)

The setup for elastic IPs on each NAT gateway to the IG, as well as the setup for public IPs

![image](https://user-images.githubusercontent.com/93015308/163280154-d78ba060-2abc-4c1b-a74a-753e32e2ad3f.png)

Default route out for each zone, applied as one. This is explained in greater detail later.

![image](https://user-images.githubusercontent.com/93015308/163280393-4a8fce8f-9cb2-4e4b-9748-32f0741ac2f0.png)

This applies the above to all (public) zones to be setup initially.

![image](https://user-images.githubusercontent.com/93015308/163280506-4125819d-5fae-4361-a507-8fc7952a4aa8.png)

Now for the private subnets.


# Routes
The other component of the network YAML file is the route settings. While it also sets up the physical (well virtual devices masked as physical devices) devices, it also preprograms them with their settings. Of chief importance is how network traffic is handled. Our internet gateway is given 0.0.0.0/0 as it’s destination block. This is router-speak for “Anything routed outside our network goes through me, and out this port” since this is likely a device that doesn’t even physically exist, there probably isn’t an actual port. Moving deeper in the NAT gateways are also programmed this way, for the same reasons. 
For networks that is it. The routes for devices will need to be set up to mesh with these parameters as well, but while they will be hooked up to the network, they will be covered in a different YAML.

#	Servers and security groups
High level description

This YAML file sets up the servers, and also applies security group settings to them. It establishes the base image (assuming we have blank slate computers this is everything that is initially installed.) and specifies the key pair to be used to monitor and administrate them. In our example it opens TCP ports 80 and 22 ingress, and 0 thru 65535 egress through our Web server. As egress ports are dynamically assigned this may be acceptable, though it will need to be something kept in mind if we wish to do our own firewall work later. As set up this means that the security group set up for the web server allows webpage connections in as well as SSH.
Our Load balancer security group allows port 80 in and out, the elastic load balancer imports settings from the this group, and applies the settings across both. A listener is set up for port 80 and we have a metric for health checks set up for the group. This will allow us to view the state of the group.

#	Database and storage
High level description
This YAML will set up the database server, which will be storing information for our actual app. The only irregularity of this portion of the entire operation is that all availability zones will point to a single server sitting in our first zone. There are a few reasons for this, which likely were not covered in class, chief among these is how databases work. Generally, in order to have a fast and correct database you need to lose one of those two words. A database can be fast, but may be incorrect, or it may be correct but be slow. This all depends on usage. A good example of how this looks like in real usage is MMOs. They have massive databases of players, but must have incredibly high availability and very low tolerance for rollbacks. To ensure this is the truth they shutdown the servers often times weekly.  So we will have a primary and a backup or secondary, the primary will be connected to by the other network and it will need to have a route from the zone 2 web server to the zone 1 database, and from the zone 1 database to the zone 2 database directly. Additionally in the case of a failover we will want the zone one web server to access the zone 2 database seamlessly and quickly.
