# Global Server Load-Balancing, Failover, Geo-Targeting and D/DoS Protection using only existing simple DNS features

In this document, this technique will be referred to as DNS/GSLB.

## Disclaimer
1. This document represents my views and does not reflect on my employer in any way whatsoever.
2. DNS/GSLB was not invented by me, but I have been unable to find the original inventor. DNS/GSLB is rarely discussed, hence this document. I have known about and used DNS/GSLB for some 20 years.
3. I have really bad dyslexia, so there will be spelling errors & typos. It does not effect my ability to conceptualise complex systems. In fact there is evidence the [opposite is true](https://inthemindseyedyslexicrenaissance.blogspot.com/).

## Preamble
There is an underlying assumption that you are providing a client facing service, from a DNS host name, using one or more POPs or vPOPs, optionally at geographically or globally dispersed locations.

If you are reading this document, it will be because you are interested in providing Load-Balancing, Failover, Geo-Targeting and D/DoS Protection across those POPs.

I will talk about the service answering client requests, but DNS/GSLB is reasonably independent of the exact nature of the service. However, it does work better
with a series of short client connections, each preceeded with a host name resolution, making it reasonably ideal for an HTTP/HTTPS service, e.g. a rest/api, but possibly less ideal for services that use long held connected sessions.

That said, the problem of providing these features on a service that uses long held connected sessions is reasonably generic.

## Definitions

| Term | Meaning |
|------|---------|
| POP | Point-of-presence - instances where you have servers providing the service, e.g. a data centre with a rack of servers or cloud services provisioned in a specific data centre. |
| vPOP | Locations where you have equipment that connects clients to the service, but doesn't provide the service directly. |
| Failover | The ability to continue the operation of the service when one or more POPs fail, either fully, partially or intermittently |
| Load balancing | The ability to spread client requests across the POPs providing the service, ideally where the service operator has some control over how the load is spread. |
| Geo-Targeting | The ability to make client requests go to the POP that is network-closest to the client, i.e. the location with the lowest latency between the client and the service. |
| D/DoS Protection | The ability of the service to continue normal operation for a significant proportion of clients, preferably all, despite being subjected to a D/DoS Attack |

For the purposes of this document, the term `POP` will include `vPOPs` unless explicitly stated. A vPOP might be something like a router/firewall/VPN, installed at a remote location, that connects to a central service hub over a private circuit of some sort.
This allows the service provider to scrub traffic locally at the vPOP while installing minimal hardware at the remote location.


## Introduction

A technique for doing failover in DNS, known to many people, is simply to allocate multiple IP Addresses (v4 and/or v6) to the same host name and rely on the client to perform failover operations.

This usually takes the form of the client experiencing a timeout, or other kind of connection failure, then either trying other IP Addresses it was given, or the client asking its resolver for the IP Address again and relying on the resolver to give the client a different answer.

This has obvious flaws, which is probably the reason it is rarely used, especially for interactive clients used directly by end users, e.g browsers. Although it can still be useful for scenarios like email delivery, where delays in the order of a few seconds to a minute are less consequential.

Instead people tend to rely on specific software and / or services to provide these functions, often employing multiple resources, as it can be rare to find any one resource that provides all four functions.

However, there is a way in which you can have fail-over, load-balancing, geo-targeting and D/DoS protection using only DNS. This has the huge advantages of not introducing more points of failure, has little to no additional cost, uses off-the-shelf software and mostly uses the systems & services you were going to run anyway (mostly!).

This is because resolving DNS names already has these features built in, so we can use a specific DNS configuration to add these features to any service.


## The Proposal

### Some Background on how DNS Resolvers Work

When resolving a DNS name, having received a delegation to a sub-domain's name servers, most resolvers will usually
1. Send all queries they want to resolve to more than one of the Name Servers, or at least very quickly failover to alternate name servers when the first one does not respond.
2. Return to the client, & cache the answer, that comes from the Name Server that was the first to get a response back to the resolver.
3. Remember which Name Servers can answer the fastest, but periodically recheck that.


### What changes

We can use this functionality to create a configuration that provides fail-over, load-balancing, geo-targeting and D/DoS Protection using only DNS. We do this by installing off-the-shelf DNS software at each POP that hosts a service and, in the parent zone file, changing the label for that service from a host name (a label that only has `A` or `AAAA` records) to a delegated subdomain (a label that only has `NS`/`DS` records), here referred to as a “service subdomain”.

Then each POP becomes directly responsible for resolving the host name of the service and will only resolve the service name to its own IP Address. The ability and speed with which the POP provides that DNS resolution will directly determine the amount of request traffic (if any) that POP receives for that service, as each IP Address will only point to the POP that gave the answer out.

DNS will then take care of using the IP Address that arrived first, which also means only from POPs that are currently up and the fastest to get an answer back to the resolver. Thus sending clients to the POP that is most capable of taking their requests, because it is up and it is either the nearest (lowest latency) or it is the least busy, so responded first.

The DNS service on each POP can also be separately taken down, leaving the user facing service is still running, but not attract traffic from the public. This can be extremely useful for maintenance and means that full Q/A checks can be run on the host before restarting the DNS service to bring it back into service, e.g. after a scheduled upgrade.

DNS is extremely resilient when some authoritative name servers do not respond.

The DNS at the POP should be running either directly on the host(s) that are providing the service, or on a load-balancer (e.g. `nginx`) that acts as the Internet facing service provider.


NOTE: Technically the client is not being sent to the POP nearest itself, but the POP that is nearest the resolver that the client is using. However, it is extremely common practice for clients to use resolvers that are close to them, i.e. hosted at their ISP, their organisation or a public DNS service that uses anycast, e.g. `8.8.8.8` - this is because DNS underpins all internet activity, so latency in responses from a DNS resolver can adversely affect the performance of all internet access, making a connection that is otherwise fast, seem slow.

This means that the POP nearest the client, and the POP nearest the client's resolver, are pretty much synonymous so, in this document, they are treated as the same thing.


## Additional Functionality

With special integration between the DNS server and the application running at the POP, the DNS server could be controlled so that the exact amount of client request traffic each POP receives can be tuned by how quickly (or if at all) the POP's DNS service responds to the DNS request to resolve the IP Address. This functionality would require something beyond what may be available in off-the-shelf DNS software, although simple throttling can be done with a one line firewall rule.

This would allow very loaded POPs to dynamically & automatically shed load to other POPs. Obviously, this assumes spare capacity somewhere in the system, but if the system lacks enough capacity there's nothing DNS can do to solve that. Although slowing the rate of DNS responses, to reduce load, may still result in a better outcome for the hosts than taking excess client traffic and crashing the server!



## How this Provides Load Balancing

The number of client requests a POP receives will be directly proportional to the number of DNS requests it answers. This means the service operator has extremely fine control over the number of requests that goes to each POP.

Using something as simple as a single line firewall rule, the service operator can throttle the number of DNS queries the service sees & answers, so controlling the amount of load that POP gets, with any 
exceeds load shed to other POPs.


## How this Provides Failover

A POP, where the server or connectivity is down, will stop answering DNS queries, so will stop receiving client requests within the timeout set by the service operator (see the example below).

Where the failure is intermittent, e.g. the connectivity has become intermittent or lossy, the number of client requests received will be reduced in the ratio of the number of DNS requests that
do not get answered, with clients being sent to the next nearest POP instead.

If the fault resolves itself, service will automatically return to normal, without any operator intervention.


## How this Provides Geo-Targetting

When the client's DNS resolver resolves the host name of the service, it will accept the answer from the POP that gets its answer back to the resolver first. Unless its heavily loaded, this will be the network-nearest POP
and will answer with the IP Address of that POP, thus sending the clients of that resolver to the network-nearest POP.

From then onwards the resolver will prefer to use this POP for resolving the host name of the service as it is the fastest to respond. But if the nearest POP starts to fail to respond, or responds slower, the resolver will start to prefer the next nearest.

Resolvers will also periodically check the response time of all POPs to ensure it is still using the fastest.


## How this Provides D/DoS Protection

When an attacker targets the service, if they start by resolving the service then launching an attack against the IP Address they have received, they will only be attacking one of many POPs.

While the POP is being attacked, it will be difficult for that POP to get DNS responses back to the client's resolvers, so the clients resolvers will get responses from other POPs first and send the clients to the other POPs, that are clear of attack. Therefore, in this scenario, one POP gets attacked and real clients get sent to the POPs that are free from attack.

If the attacker re-resolves the host name periodically, then they will simply end up switching their attack from one POP to another, and the service will react by switching the clients to the POPs that are not getting attacked.

If the attacker tries to attack all POPs by continuously re-resolving the service host name, this will be relatively easy to detect & mitigate against, e.g. give the attacker a localhost IP Address as the answer.

If the attacker uses a distributed bot-net and resolves the name from multiple places on the internet to get multiple different answers, they will then be able to attack all POPs simultaneously,
but the load of the attack will be spread across all POPs.


## An Example Configuration - The Old Way

For the site `www.exmaple.com` hosted on three servers on the IP Addresses `172.16.0.1`, `172.17.0.1` and `172.18.0.1` the old way of getting DNS to provide failover would be to have a DNS configuration like this

    $ORIGIN example.com.
    $TTL 86400
    www IN A 172.16.0.1
    www IN A 172.17.0.1
    www IN A 172.18.0.1

Although this example uses private (RFC1918) IP Addresses DNS/GSLB works equally well in public and private networks.


## An Example Configuration - The New Way

### The Parent Zone File – example.com

    $ORIGIN example.com.
    $TTL 86400
    www IN NS ns1
    www IN NS ns2
    www IN NS ns3
    ns1-www IN A 172.16.0.1
    ns2-www IN A 172.17.0.1
    ns3-www IN A 172.18.0.1

With this change, `www.example.com` has changed from being a host name to being a delegated sub-domain.

Then on each of the three servers we must now install an off-the-shelf authoritative DNS server package which is configured to only serve the domain name `www.example.com`. The domain data will have a single IP Address record where each server returns only its own IP Address as the answer.

So, the zone file for `www.example.com` on `172.16.0.1` will look something like this

    $ORIGIN example.com.
    $TTL 86400
    www IN SOA . . 1 1 1 1 1
    www IN NS ns1-www
    www IN NS ns2-www
    www IN NS ns3-www
    www 10 IN A 172.16.0.1

The same additional software and configuration will also be applied to hosts 17 & 18, but with their own IP Addresses configured into the domain data for `www.example.com`.

It is important that each of the hosts lists all the other hosts in the NS records (in the service subdomain) as these are the authoritative records, so will be preferred over the `NS` records from the parent zone, which have the status of GLUE records.

Although, in our example, each host only serves out a single IPv4 Address, they could serve out any number of IP Addresses (v4 and / or v6). The important thing about DNS/GSLB is that the address(es) point to the POP that is giving them out.

The TTL on the IP Address record(s) will determine how long it will take for clients to failover to a different POP, when the POP they had been talking to dies.

Clients who are using a resolver that has this IP Address in cache will have to wait for up to this TTL to timeout before their resolver will re-request the IP Address from the remaining POPs and receive an answer from a POP that is still up. Resolvers that do not have the IP Address of a failed POP in cache will be directed to a working POP right away.

When the DNS service on a POP comes back up, it should start receiving traffic right away, so consideration may need to be given to the order services are brought back up during a reboot. That is, the DNS Service should only be brought back up when the client facing services are ready to start answering client queries again.


## Disadvantages

1. It will take slightly longer to resolve the IP Address of the service, but this disadvantage will be balanced out by the fact that users will be sent to the POP with the lowest latency, so all traffic related to the service should arrive to the user as fast as physics & the server performance allows.
2. Setting up DNSSEC is even more complicated!
3. If you are using a firewall in front of (and/or on) the service hosts, you will need to open access to port 53 for TCP & UDP to allow DNS to work. If you are using connection tracking you may need to extend the number of slots & tune the timeout for UDP tracked connections as DNS can use a lot of slots fast. A UDP conntrack timeout of 12 seconds is common for DNS.


## DNSSEC and DNS/GSLB

Setting up DNS/GSLB for DNSSEC is definitely more complex than for a traditional centrally controlled zone file.

The best options are probably
1. Sign the service subdomain multiple times, once with each IP Address of each POP, then distribute the pre-signed zone files to the POP. This way there is a single `DS` record for the service subdomain.
2. Sign each POP's zone file separately, with separate keys, then place a `DS` record in the parent zone for every POP (i.e. each set of keys used).
3. Don't DNSSEC sign the service subdomain!


## Does it Really Work?

I have known about and used DNS/GSLB for some 20 years. I have never had any issues with it. For 10 years I was responsible for all the servers and services at the dot-IO Domain Registry where I used DNS/GSLB.

We had two POPs in London & a vPOP in Ashburn. DNS/GSLB was used on a wide range of services, for all its benefits, to distribute load over these locations.


## It Could be so Much Easier

One way to make DNS/GSLB much easier to set up and run would be to have the very basic DNS server functionality necessary at the POP built into the software that is providing the client service.

The amount of DNS work it needs to do is extremely minimal, so adding a module to an existing application, like `nginx`, should be a relatively straight forward task.

This would obviate the need for adding DNS software to the POPs and should make configuration much easier, as a lot of the DNS response information can be hardcoded, and pre-created at start-up.

This would have zero effect on any of the existing functionality of the software, but mean all the user needs to do is make the appropriate changes to the parent zone and some minor changes to the config at the POPs.

Although support for modern DNS functionality like `EDNS0` and DNS Cookies would be “nice to have” it should not be necessary.

If the `nginx` server was hosted on a cloud service like AWS, it may be necessary for the module to also support RFC2136 Dynamic DNS Updates to change the IP Address given out, as cloud services often work on dynamically allocated IPs.


## DNS/GSLB vs Anycast

DNS/GSLB has some obvious similarities with Anycast. So it would be amiss to not discuss the relative pros and cons of each.

Each technique uses a separate signalling protocol to flag to clients whether a particular POP is available for service or not, so drawing request traffic to that POP, or not. Anycast uses BGP, DNS/GSLB uses DNS.

### Some Notable Features to Understand Anycast

1. Anycast will pick the “cheapest” route from client to servcie. This may be the same as the route with the lowest latency (aka “network nearest”), but often is not. Where as DNS/GSLB will always pick the usable POP with the lowest latency.

    Typically Anycast will route based on the lowest number of hops, so is highly dependant on peering agreements. Differential financial cost may be involved between peers, influencing routing decisions. Carriers will often count their entire internal network as zero hops. All these factors can mean the “cheapest” route is not the same as the lowest latency POP.

2. Although, to the service provider, the service appears to be serviced by a number of POPs, to the client there is only ever one single POP – the one they are being routed to. To the client, all the other POPs do not exist and are inaccessible, until the BGP is withdrawn from their “cheapest”.

    Therefore, when using anycast, care must be taken to ensure the BGP route is withdrawn when the service ceases to be viable at any one POP. However, this also means, if they POP is attack, the clients being sent there will be adversely affected.

3. Anycast is not actually a specific internet feature, but is fooling the internet into thinking that two or more identical servers are a single multi-homed server. This means the internet will assume that packets going to server can take any known route but, under anycast, this can mean packets end up at different servers - this can cause technical issues - which means anycast is not an option for all protocols.


### Comparing Anycast with DNS/GSLB

| Feature | DNS/GSLB | Anycast
|-----------|---------------|-----------
| Load-balancing | Can be fine tuned to the operator's exact specification | Anycast may spread the load over the POPs, but the operator has little control over how this happens as it may depend on 3rd party peering agreements |
| Failover | Generally this is automatic | The operator is responsible for ensuring this happens, which can be done with an automation |
| Geo-targeting | Clients are always directed to the usable site with the lowest latency | Yes, but some what haphazard and unpredictable, see above
| Anti D/DoS | Mostly the same | Mostly the same |

With Anti D/DoS the functionality is quite similar, in that attack traffic will probably either be all drawn to one site, or evenly spread across all sites – depending on the exact attack profile.

However, if a single POP gets attacked, with anycast, legitimate clients (going to that POP) will inevitably suffer, as they can only ever see their “cheapest” POP, but DNS/GSLB clients will probably be automatically sent to the next-nearest site. With Anycast, if the BGP is taken down, to move the clients to the next-nearest POP, the attack traffic will move with them, where as (with DNS/GSLB) separating attack traffic and legitimate clients is possible.

### Pros & Cons

Anycast is well understood, widely used and widely recognised. Staff with skills in anycast are therefore generally available. DNS/GSLB is not widely used, so few people will recognise it when they see it and people with detailed DNS technical skills are becoming increasingly harder to find.

Anycast hosting will require a specialist hosting provider who can support BGP. This is becoming easier to find, but there will always be relatively limited choice and a higher cost. DNS/GSLB only requires standard DNS, so should work with almost any hosting provider.

Under some conditions, anycast can end up seeing two routes as exactly equal cost and so send packets, from the same client, equally to both POPs. This is rare, but it can happen. When this happens it breaks TCP. This will never happen with DNS/GSLB as the client gets “locked in” to one POP for the duration of a TCP connection and/or the period of the TTL.
