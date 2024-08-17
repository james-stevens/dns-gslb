# Near Live Global Server Load-Balancing, Failover, Geo-Targeting and D/DoS Protection using only existing DNS


## Introduction

The technique, for doing failover in DNS, known to many people is simply to allocate multiple IP Addresses (v4 and/or v6) to the same host name and rely on the client to perform failover operations.

This usually takes the form of the client experiencing a timeout, or other kind of connection failure, then either trying other IP Addresses it was given, or re-resolving the client asking its resolver for the IP Address again and relying on the resolver to give the client a different answer.

This has obvious flaws, which is probably the reason it is rarely used, especially for interactive clients used directly by end users, e.g browsers. Although it can still be useful for scenarios like email delivery, where delays in the order of a few seconds to a minute are less consequential. 

Instead people tend to rely on specific software and / or services to provide fail-over, load-balancing and geo-targeting functionality.

However, there is a way in which you can have fail-over, load-balancing and geo-targeting using only DNS. This has the huge advantages of not introducing more points of failure, as little to no additional cost, uses off-the-shelf software and mostly uses the systems & services you were going to run anyway (mostly!).

## The Proposal

### Some Background on how DNS Resolvers Work

When resolving a domain name, having received a delegation to a sub-domain's name servers, most resolvers will usually
1. Send all queries they want to resolve to more than one of the Name Servers, or at least very quickly failover to alternate name servers when the first one does not respond.
2. Return to the client & cache the answer that comes from the Name Server that was the first to get a response back to the resolver.


### What changes

We can use this functionality to create a configuration that provides fail-over, load-balancing and geo-targeting using only DNS. We do this by installing off-the-shelf DNS software on each server that hosts a service and, in the parent zone file, changing the label for that service from a host name (a label that only has `A` or `AAAA` records) to a delegated subdomain (a label that only has `NS`/`DS` records), here referred to as a “service subdomain”.

Then each host becomes directly responsible for resolving the host name of the service and will only resolve the service name to its own IP Address. The ability and speed with which the host provides that DNS resolution will directly determine the amount of traffic (if any) that host receives for that service, as each IP Address will only point to the host that gave it out.

DNS will then take care of using the IP Address that arrived first, which also means only from hosts that are currently up and the fastest to get an answer back to the resolver. Thus sending clients to the host that is most capable of taking their requests, either because it is the nearest (lowest latency) or because it is the least busy, so responded first.

The DNS service on each host can also be separately taken down, leaving the user facing service is still running, but not attract traffic from the public. This can be extremely useful for maintenance and means that full Q/A checks can be run on the host before restarting the DNS service to bring it back into service, e.g. after a scheduled upgrade.

DNS is extremely resilient when some authoritative name servers do not respond.


NOTE: Technically the client is not being sent to the host nearest itself, but the host that is nearest the resolver that the client is using. However, it is extremely common practice for clients to use resolvers that are close to them, i.e. hosted at their ISP, their organisation or a public DNS service that uses anycast, e.g. `8.8.8.8` - this is because DNS underpins all internet activity, so latency in responses from a DNS resolver can adversely affect the performance of all internet access, making a connection that is otherwise fast, seem slow.

This means that the host nearest the client, and the host nearest the client's resolver, are pretty much synonymous so, in this document, they are treated as the same thing.


## Additional Functionality

With special integration between the DNS server and the application running on the host, the DNS server could be controlled so that the exact amount of client request traffic each host receives can be tuned by how quickly (or if at all) the host respond to the DNS request to resolve the IP Address. This functionality would require something beyond what may be available in off-the-shelf DNS software.

This would allow very loaded servers to shed load to other servers. Obviously, this assumes spare capacity somewhere in the system, but if the system lacks enough capacity there's nothing DNS can do to solve that. Although slowing the rate of DNS responses, to reduce load, may result in a better outcome for the host than taking excess client traffic and crashing the server!



## An Example Configuration - The Old Way

For the site `www.exmaple.com` hosted on three servers on the IP Addresses `172.16.0.1`, `172.17.0.1` and `172.18.0.1` the old way of getting DNS to provide failover would be to have a DNS configuration like this

    $ORIGIN example.com.
    $TTL 86400
    www IN A 172.16.0.1
    www IN A 172.17.0.1
    www IN A 172.18.0.1

Although this example uses private (RFC1918) IP Addresses the technique in this document works equally well in public and private networks.


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

Then on each of the three servers we must now install an off-the-shelf authoritative DNS server package which is configured to only serve the domain name `www.example.com`. The zone will have a single IP Address record where each server returns only its own IP Address as the answer.

So, the zone file for `www.example.com` on `172.16.0.1` will look something like this

    $ORIGIN example.com.
    $TTL 86400
    www IN SOA ns1-www.example.com. Hostmaster.example.com 1 1 1 1 1
    www IN NS ns1-www
    www IN NS ns2-www
    www IN NS ns3-www
    www 10 IN A 172.16.0.1

The same additional software and configuration will also be applied to hosts 17 & 18, but with their own IP Addresses configured into the zone file for `www.example.com`.

It is important that each of the hosts lists all the other hosts in the NS records (in the service subdomain) as these are the authoritative records, so will be preferred over the NS records from the parent zone, which have the status of GLUE records.

Although, in our example, each host only serves out a single IPv4 Address, they could serve out any number of IP Addresses (v4 and / or v6). The important thing about this technique is that the address(es) point to the host that is giving them out.

The TTL on the IP Address record(s) will determine how long it will take for users to failover to a different host, when the host they had been talking to dies.

Users who are using a resolver that has this IP Address in cache will have to wait for up to this TTL to timeout before their resolver will re-request the IP Address from the remaining hosts and receive an answer from a host that is still up. Resolvers that do not have the IP Address of a failed host in cache will be directed to working hosts right away.

When the DNS service on a host comes back up, it should start receiving traffic right away, so consideration may need to be given to the order services are brought back up during a reboot. That is, the DNS Service should only be brought back up when the client facing services are ready to start answering queries again.


## Disadvantages

1. It will take slightly longer to resolve the IP Address of the host, but this disadvantage will be balanced out by the fact that users will be sent to the host with the lowest latency, so all traffic related to the service should arrive to the user as fast as physics & the server performance allows.
2. Setting up DNSSEC is even more complicated!
3. If you are using a firewall in front of (and/or on) the service hosts, you will need to open access to port 53 for TCP & UDP to allow DNS to work. If you are using connection tracking you may need to extend the number of slots & tune the timeout for UDP as DNS can use a lot of lot fast. A UDP conntrack timeout of 12 seconds is common for DNS.


## DNSSEC and this Technique

Setting up this technique for DNSSEC is definitely more complex than for a traditional centrally controlled zone file.

The best options are probably
1. Sign the service subdomain multiple times, once with each IP Address of each host, then distribute the pre-signed zone files to the hosts. This way the DS record is the same for all three hosts.
2. Sign each host's zone file separately, with separate keys, but place a DS record in the parent zone for every host (i.e. each set of keys used).
3. Don't DNSSEC sign the service subdomain!


## It Could be so Much Easier

One way to make this whole process much easier would be to have the very basic DNS server functionality required built into the software that is providing the client service.

The amount of DNS work it needs to do is extremely minimal, so adding a module to an existing application, like `nginx`, should be a relatively straight forward task.

This would obviate the need for adding DNS software to the hosts and should make configuration much easier, as a lot of the DNS response information can be hardcoded and pre-created at start-up.

This would have zero effect on any of the existing functionality of the software, but mean all the user needs to do is make the appropriate changes to the parent zone.

Although support for modern DNS functionality like EDNS0 and DNS Cookies would be “nice to have” it should not be necessary.

If the `nginx` server was hosted on a cloud service like AWS, it may be necessary for the module to also support RFC2136 Dynamic DNS Updates to change the IP Address given out, as cloud services often work on dynamically allocated IPs.

