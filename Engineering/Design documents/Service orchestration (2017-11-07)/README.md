|              |                       |
| ------------ | --------------------- |
| Author       | Kaspar Schiess (Pryv) |
| Version      | 1 (7.11.2017)         |
| Distribution | Internally            |

# Why Service Orchestration Now

## Introduction

As part of the high frequency implementation in Pryv, we're installing and exploring a tool for orchestrating our servers and releases. Since there is quite a bit of effort involved in doing so, we would like to write down our reasons for doing so for posterity. 

## Installing and Upgrading - In a Nutshell

We ourselves maintain at least 3-4 installations of Pryv in various locations. A part of selling Pryv will always be assisting with installation. Some of these installations will have to scale to large sizes; some of them will start off their life on servers owned by Pryv. 

Installing and updating Pryv has become easier to do during 2017. Installing Pryv now involves these tasks: 

* Creating configuration for the client/project. (~ 2h)
* Creating VMs for installation, via Terraform & Ansible (~ 30' per VM)
* Uploading configuration, login to Bintray, launching configuration. (~ 30min per VM)
* Test (~ 1h)

This brings us (and our clients) to 6h installation time for a Pryv IO, from scratch (3h + n*1h). Updating a Pryv involves: 

* Updating configuration. (~ 30')
* Uploading configuration to each server, restarting and cursory verification. (~ 30' per VM)
* Test (~ 30min)

Updating a Pryv installation takes therefore 2.5h for an installation with 3 machines (1 + n/2). 

## Scalability Story - The Big Picture

While it is true that Pryv is conceived to scale out to 1000s of servers, no one has done so. Even if presented with the problem, customers have chosen not to scale. We were given 'cost' as a reason for this. 

Where does this cost come from? Not only from the cost of the virtual machines involved. Our main cost driver is currently the complexity of an installation. For a big Pryv installation, you need to: 

* Create machines at scale. 
* Distribute and run Pryv. Either manually (big n!) or automatically (expertise!).
* Scale up and out.
* Monitor the servers involved. 
* Centralise logs by datacenter (privacy!). 
* Update all servers every few months, both OS and Pryv install.
* Backup and Restore.
* Replication and HA.

I am going to address each of these points individually, comparing the problem to our current offer. 

_Create machines at scale, Distribution and Update_ – Pryv installs via docker/docker-compose. This would enable a really skilled administrator to tackle the problem of distribution and update. Creating machines at scale - while not entirely trivial - can be considered a solved problem (again for a skilled administrator). We consider our efforts here incomplete; while it is true that installation has become simple, installation at scale is still hard. What we should do: 

* Publish recipes for machine creation (terraform) and distribution at scale.
* Ensure compatibility with several orchestration products, such as Docker Swarm, Kubernetes, Nomad and Amazon EC2. 
* Instead of distributing configuration among many files make sure that configuration is as simple as possible. Maybe even configure the cluster centrally? 

_Monitoring_ – We have implemented a system using Netdata, InfluxDB, Kapacitor, Chronograph and Grafana for our servers. We are capable of consulting on using Collectd, Telegraf, Librato, Datadog and possibly others. What is missing here are relevant performance metrics from inside Pryv, forwarded to a StatsD server. Our offering here is quite good, missing are: 

* Recipes for installation of these products
* Integration of Pryv performance metrics into external systems (via StatsD). 
* Internal dashboard and monitoring for a batteries-included kind of installation. 

_Scale Up and Out_ – Scaling up is currently near impossible (1.2). Scaling out is hard; configuration files on the registry must know about all machines at start time; so adding a machine will mean restarting the registry. We have added basic features for scaling out in 2017, but that's not enough. Here's what we want to do: 

* As part of HF Pryv, allow scale up to the limits of a single machine. 
* Render configuration dynamic; adding cores should not mean restarting the Registry. 
* Be compatible with the service orchestration systems mentioned above, be able to consult on implementing such systems. Nobody in their right mind will create 100s of servers by manual labor (since you always do things at least twice).

_Monitor the servers_ – Monitoring a big installation is the only way to keep it running. We don't currently offer much support here. What we should do: 

* Write recipes for using any of the usual stacks. This might just be a list of articles we curate, since many such blog posts exist on the interwebs. 
* Output performance metrics from within Pryv. 

_Centralise Logs_ – Our log files currently don't even help us debug our issues. They cannot in good spirit be called an audit log either. Security problems will go unnoticed. We should: 

* Provide recipes on how to centralise logs per datacenter and make sure that important log entries aren't lost in the volume. 
* Add structure to logs. Distinguish better between debug information, audit information and security information. Make sure that all the categories really solve a problem for a stakeholder. Avoid logging for loggings sake. 
* Log in a structured format (JSON?) to allow (correct) parsing and processing. 

_Updates_ – The effort required to update an installation currently approaches that of installing for the first time. On one hand we'd like to be able to push security updates every few months - on the other hand we currently risk confronting our clients with a subpar experience. This would be improved by: 

* A clear story (recipe) on how to deploy at scale (out). We should exercise this with pryv.me as well, making it convenient for us (and by extension, for our customers).

_Backup and Restore_ – We back up pryv.me and pryv.li on servers in the same datacenter. While sufficient in this context there might be more to this. No list of ideas currently. 

_Replication and HA (high availability)_ – Here's what we'd like to have: 

* Allow replication of either a complete server, a single user or a subset of data for a single user to another server. This would be a leader/follower-type setup where you can read data back from the follower, but you only write to the leader. 
* A real high availability design for all components. Replication might be part of this, with automatic failover to a replika. 

## Why Orchestration Now?

Much of our daily routine is dominated by the topics outlined above. We're either installing / updating our own installations or discussing with clients around the same topics. If we can elevate this routine to a different level, we should. 

The Devops movement has discovered that automating things doesn't just save time, it reduces perceived complexity of tasks. More tasks become possible to do because they fit better into the complexity budget of a single developer. This is a strong effect that is hard to reason about. 

More concretely, by attacking orchestration now, we can deploy intermediate builds of High Frequency Pryv (1.3) to a server and test it there. Deploying intermediary stages of our development process becomes possible; this makes our testing better and makes our work visible. 

We're at a point where we can reap benefits along multiple dimensions at once. The only thing we need to do is invest in a task earlier than we would have according to plan - orchestration would have come at the end of HF Pryv, not at the start. The savings we can make outweigh the cost of changing our schedule. 
