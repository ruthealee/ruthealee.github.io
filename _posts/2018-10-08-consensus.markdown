---
layout: post
title: "Brief Notes on Raft & Consensus Algorithms"
date: 2018-10-08
categories: [linux]
---
I've been thinking a bit about Consensus Algorithms recently. Most people who have some CS background are pretty familiar with the Byzantine General Problem, and consensus algorithms are an attempt to solve for this essentially. Since my background isn't in CS this was something I was tangentially aware of but hadn't had a chance to dive into. A recent project that dealt with Splunk's Search Head Clusters lead me to research some of the underlying architecture in more detail and take some notes.

Rather than look at the algorithms as standalone instead I'm looking at systems that utilise them. This allows for  more practical understanding of real world implementations.


##Byzantine General Problem
The Byzantine General problem is particularly interesting to me since I have a bit of a background interest in military history, particularly around early pre-biblical Near Eastern and Medieval time periods, as well as early cryptography. That means I spent quite a bit of time thinking about interceptions of communications just not in a computing context! Honestly there's no point rehashing it here, it's very well explained by [wikipedia](https://en.wikipedia.org/wiki/Byzantine_fault_tolerance#Byzantine_Generals'_Problem).
Fundamentally though consensus algorithms are all trying to solve the same issue, how do you reach a consensus when you may lose contact with some of your nodes, and further some nodes may be lying about their status.



## Paxos
One of the most known consensus algorithms is Paxos. I actually hadn't really come across it before. Previously I'd been working with Redhat Clustering and so hadn't had to dive into alternative systems.
I read through Leslie Lamport's paper [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) and learned a fair bit. If this was the simple version, I do not want to read the complicated version! I think this is roughly the idea, at least when deployed in terms of a clustered set of worker servers that require a single leader to batch work out to them or co-ordinate in some way.

* When a node proposes a value (let's assume the value it is proposing is it's captaincy or primacy as leader of a cluster) it assigns an incremented number as it's proposal number. For this example let's say that Node 1 proposes itself as leader of a cluster of nodes, and it assigns proposal number 1 to that request. It then sends that proposal to all the other nodes it is aware of.
* The other nodes then look at the proposals they have received, and they pick the proposal with the highest proposal number value, and they respond back saying 'we will ignore all proposals with numbers less than this (1 in our case)'.
* Once a quorum of responses have been returned to the proposing node, it can now proceed and send out an accept request that states it's the proposer who's value is the highest and requests acceptance of this fact from the other nodes.
* The nodes return with an accepted response and leadership is assigned.

You might guess in the above that if one node receives the initial proposition with value 1, and then subsequently doesn't receive a accept request that it will then go ahead and try to propose itself. Indeed it will, and it would number its proposal 2. The same pattern would occur, and depending on th epoint in the acceptance chain of the other nodes, they may end up in a situation where they end up turning down the accept request from node 1, since they already received a competing proposal with a higher proposal value. Since leadership isn't assumed until a quorum of nodes have returned the accepted response the first proposer (our Node 1) will simply switch and instead accept the higher valued proposal from our second node.

## Raft Consensus
Splunk SHC uses RAFT. You can find basics here: https://raft.github.io/ and the whitepaper lives here: https://raft.github.io/raft.pdf
The white paper is very good and is recommended reading to better understand how election happens.

However, here is a very brief rundown:

* Every node starts up as a 'follower'
* Followers receive an RPC call (aka heartbeat) from the leader (the captain)
* Captains are elected for a set 'term' or period, which is monotonically increasing (i.e. it relies on internal kernel timers not external timesources to protect from time skew issues)
* The captain communicates with its follower nodes, and will monitor its term as captaincy. When the term expires it will offer itself for re-election. If it achieves quorum it will extend its term and continue. A captain will only step down from captaincy to follower if it finds itself unable to communicate with a number of its followers. This will usually happen in conjunction with followers realising they have no contact and offering themselves as candidates for election anyway.
* A node remains a follower until it stops receiving a leader heartbeat call. Once a term ends, the leader reverts to follower thus meaning that all the nodes stop receiving the leader heartbeat. *Alternatively the leader may fail to communicate prior to the term expiring, which will also result in the nodes no longer receiving a heartbeat.
* When a node stops receiving the heartbeat from the leader it will go into a state of candidacy for leadership. It will call out to the other nodes and request to be voted the leader. If you're thinking 'but that will cause split votes' you're correct! To deal with this raft introduces some arbitrary jitter into the nodes candidacy requests so they don't all run at the same time. See the whitepaper for exact details.
* The other nodes respond to the election request and confirm election. Election is assigned once a node receives enough votes to achieve quorum. Quorum is set as >50%. So, in a 5 node SHC, 3 members.
* The new captaincy is assigned and the term is reset and begins to monotonically increment once again.

As you can see from the above knowing this helps us to understand a few things such as: Why some nodes seem to retain captaincy basically forever (they're the ones initiating the new election and extending their term). We can also see how the system is designed to deal with network partitioning and other failures. Bear in mind that quorum must be achieved in order to adequately elect a captain, this means that dependent on the size of your cluster there may be scenarios where you find yourself without quorum and unable to elect a captain via node consensus. In the case of Splunk this is a scenario in which you would statically elect a captain as per this documentation: http://docs.splunk.com/Documentation/Splunk/7.1.3/DistSearch/StaticcaptainS

## RedHat Clustering
This is what I'd be most familiar with as it's what I worked with the longest. This is probably the simplest of these in terms of its consensus algorithm, it's simply looking for quorum based on heartbeats of nodes. Probably better to think of it as failure recognition and remediation. It can become quite aggressive about fencing what it deems to be an unresponsive node.

Here's what I know about RHCS, based mainly on RHEL 6 systems not Pacemaker implementations in 7.

* CMAN is the service that deals with node management, and there isn't necessarily a captain or leader of an RHCS cluster, rather there are nodes upon which serviecs are distributed. This means that multiple nodes can be the active node, running different services. In an ideal layout you want to distribute services across the nodes, so if you have two services on a 5 node cluster you'd like one to be active on node1, one on node2, and the remaining 3 nodes acting as passive nodes available for service failover.
* One big benefit of RHC is the ability to kill other unresponsive nodes. This is generally done over ethernet in the case of physical servers, but can also be run via other methods. Fencing provides the nodes the ability to restart each other. With that benefit however comes some additional risk in that you can find yourself in race condition where the nodes consistently reboot one another and this is deeply unpleasant.
* RHCS works on quorum similarly to the others. A node comes online and checks in with the other nodes in the cluser to see where the services are running. If a service is in a failed state the cluster will attempt to distribute it to one of the nodes. You can configure many options around this, particularly with regard to failover domains which basically limit the nodes to which a service is allowed to be distributed.
* Once a service is running the cluster manager basically simply deals with ensuring that it maintains quorum with regards to its nodes. A heartbeat service provides communication between the nodes. If any node stops receiving communication from another node it will initiate a fence protocol which utilises your defined fence method to remove the node from the cluster in the hopes it will restart correctly and rejoin.
* When it comes to quorum one nice feature of RHCS is that you can manually set the quorum value as well as increase the number of votes of any individual node. In combination with failover domains this allows for a very flexible and customisable cluster deployment, but again more complexity can lead to more problems if you misconfigure.
* Since there's no explicit leadership, merely active service runners, any changes to configurations that are made to services must be made on the active node. This is usually since in an RHCS deployment you are sharing some kind of backend storage, commonly SAN storage.


## Comparing Our Options
All the algorithms achieve pretty resilient consensus. There's a few things I like about each of them that are selling points for me.

* With RHC and Pacemaker you gain the ability to define failover domains and services that failover. This clustering solution is slightly different from Splunk's SHC approach. The downside for this is that if you configure this wrong you end up with a service that never fails over (believe me I've done this a few times!). Nice that you can build in fence methods to hopefuly recover from failures.
* With Raft consensus and Splunk's implementation I like that there is a remit to elect any node as the captain, and that generally speaking it scales easily. By adding more nodes we don't add any additional complexity and it's much clearer how we statically define the captain. RHC
- Paxos is a fully comprehensive algorithm and certainly accounts for a number of problematic scenarious but it is quite chatty. There's a large amount of proposing and accepting and it requires a higher number of acks from the nodes. In a system where you need high resiliency I think this might be a good option, but it's pretty much built for unreliable networks. If you can engineer a system where you have a dedicated network for inter-cluster communication you can probably get faster performance from a different algorithm.

In conclusion, I don't really miss RedHat Clustering administration, because it could get pretty complicated and it was really prone to trying to murder other nodes whenever there were problems with networking or even heartbeat timestamping. Paxos is pretty chatty and slow although resilient. Raft I like the most of these, which I guess is lucky since that's what I'll be working with mainly now!
