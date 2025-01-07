---
title: AWS Resilience Series - Fault Isolation Boundaries
permalink: posts/{{ title | slug }}/index.html
date: '2024-10-08'
tags: [aws, cloud, resilience]
---
![](/images/posts/aws-resilience-series-fault-isolation-boundaries/cover.jpg)

>Unlike the previous post in this series, this one will get deeper into AWS-specific nuances and all the points here here will not be directly translatable to other infrastructure providers. That said, the concept of fault isolation boundaries is broadly applicable across many types of software and infrastructure designs.

After my [last post](/posts/aws-resilience-series-availability-vs-recoverability), hopefully you're at least devoting some thought to your application's recoverability. While I tried to stress the importance of being prepared to recover your application in the event of a failure, availability is still a key trait for many applications.

For many folks there's likely a thought that high availability = run more than one instance of my application. On the surface, that's true! The more instances that are available to handle your workload means more things that have to fail before your workload cannot be handled. Perfect, so this means that optimizing availability is an exercise in adding as many instances as possible without going bankrupt, right?

Well, your infrastructure provider would be certainly thrilled with this approach...
![Popular cloud providers cast as Scrooge McDuck and his grand-nephews](/images/posts/aws-resilience-series-fault-isolation-boundaries/cloud-ducks.jpg)
(this is what I'm guessing cloud providers look like when you run lots of redundant instances without traffic demanding it)

What if we could instead be smarter about adding redundancy while stretching the utility of your infrastructure dollars? Fault isolation boundaries give us a means to ensure that we're building workloads that are "redundant enough" to mitigate entire classes of failures without being cost-inefficient.

But first, what are fault isolation boundaries?

## What's in a Name?

At a high level, fault isolation boundaries are logical divisions within your infrastructure that are intended to contain the impact of component failure. If you've ever had a conversation about "limiting the blast radius" of a potential failure in your application, you've likely implemented a fault isolation boundary to do it. AWS provides us with a number of infrastructure-level fault isolation boundaries:
- Availability Zones​
- Regions​
- Accounts​
- Partitions​
- Local Zones
- Control Plane vs. Data Plane Separation

These boundaries serve multiple purposes. Regions and local zones can enable operations in certain jurisdictions which have legal requirements around where data is processed and stored. Accounts (and to an extent, partitions) are important security boundaries which ensure that access is scoped appropriately. But how do these boundaries fit into a conversation around resilience? 

From a resilience perspective, AWS [uses them internally to roll out new capabilities in a safe manner](https://youtu.be/es9527rA_8I?si=jcm2P6h1OQZyfRvK&t=2361) so that any failed changes can not only be rolled back quickly, but also only impact a small number of users. When a new feature is being rolled out, availability zones will get the update one at a time (meeting certain success metrics at each step) until it is rolled to the entire region. More than just feature releases though, availability zones and regions in particular are designed with specific constraints to ensure they provide resilience value to us as the customer.

## Infrastructure Designed for Reliability

![Map of the world showing where AWS regions are located](/images/posts/aws-resilience-series-fault-isolation-boundaries/region-map.png)

AWS has [regions all over the globe](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/) and they're constantly working to add new ones. If you're new to AWS, it might be easy to view regions as simply a way to choose where your workloads run. Whether it's to provide lower latency by putting your workload physically closer to your users or ensuring that data stays where it is legally required to, regions dictate the physical location of your infrastructure. Regions are more than this though; they are a key resilience feature for AWS. They are deliberately designed to be hundreds of miles apart (within the same partition) such that a natural disaster impacting one region should not impact its neighbors. If one does experience a failure, each region is designed with independent control planes such that any outages should remain isolated (with some notable exceptions for [global services](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html)).

The next smaller boundary within regions are [availability zones](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/availability-zones.html) (often abbreviated AZs). In contrast to regions, availability zones are separated by just tens of miles to ensure that data replication between them remains fast. In spite of availability zones being closer together, they are designed to avoid "shared fate" failure scenarios such as localized natural disasters or power disruptions. AZs are interconnected by redundant fiber connections so data replication between them is super fast. Many applications should be able to operate across multiple availability zones within the same region without inter-zone latency concerns.

## Planar Separation and Static Stability

Now that we've covered physical infrastructure, let's talk about what it means to actually _run_ something on that. AWS services are implemented with a division between their control and data planes. The former is what allows us to create, update, or delete resource in our AWS accounts. Control planes provide the APIs that the Console, CLI, SDKs, and infrastructure as code interact with. Conversely, data planes are what provide the ongoing service by a given resource. Think of a control plane as the component that facilitates the creation of an EC2 instance and the data plane is what keeps the instance going and able to serve traffic.

This planar separation is important for a number of reasons, one of which being that it enables a concept called **static stability**. This is a pattern which aims to provide higher levels of reliability by designing applications that can recover from failure without the need to make any changes. In the case of an application that runs across multiple availability zones, static stability would mean having infrastructure ready to handle traffic provisioned and ready in each availability zone with no manual intervention needed to failover between them. In context this means that, since data planes are relatively simpler than control planes and thus less likely to fail, designing an application with static stability will help guard you against an outage when a control plane becomes unavailable.

## What does it all mean?
![Meme of Dory and Marlin from Finding Nemo saying "It's like he's trying to speak to me, I know it."](/images/posts/aws-resilience-series-fault-isolation-boundaries/nemo.jpg)

If you've made it this far, you might be thinking "this all sounds great, but what should I do this all this?" Since we set out trying to work out how to make our systems more resilient while also remaining cost effective, we can look at the fault isolation boundaries as a way to quantify risk.

Statistically speaking, failures of (or within) individual availability zones are more likely than entire regions. Fittingly, many services in AWS make it really easy to operate across multiple availability zones at once. Because of this, implementing statically stable workloads across multiple AZs in a single region is a good first step. Of course the additional infrastructure isn't free, but the added operational overhead is minimal. For many applications this might be enough, but the most critical workloads might demand multi-region operations. What's important here is that you're intentional with _where_ you're running your workloads. Running 3 redundant instances split across 3 AZs is far more reliable than running all 3 in the same AZ.

If using more AZs is good, then more regions is surely better, right? Regional service failures tend to be more rare than AZ-level outages, though they are more impactful. Unlike AZ failovers, most services do not provide an easy way to operate out of multiple regions at the same time so there are usually more moving parts to account for in a regional failover. Because of this, it's best to account for multi-region operations early in the design process so you can choose services and patterns that are conducive to this level of resilience. It's not impossible to retrofit later on, but it can require a lot more refactoring than just adding multi-AZ resilience. The operational complexity combined with the additional infrastructure needed makes multi-region resilience significantly more expensive than multi-AZ.

Think about AZs like having fire doors in a building. In the event of a fire, these doors help to prevent the fire from spreading to other parts of the building. Relative to the cost of the entire building, fire doors are a cheap way to help isolate the potential damage that a fire can cause (and help to keep occupants safer). Running in multiple regions is like constructing an entirely separate building to protect against a fire in the first. If one building burns down, you still have a second usable building, but the cost to do so is very high.

If you want to read more on this topic, I highly recommend the entire [AWS Whitepaper on Fault Isolation Boundaries](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/abstract-and-introduction.html). Even if you've been working in AWS for a long time, it is a great resource to dive deeper into how various AWS services are built for resilience.

In my [last post](/posts/aws-resilience-series-availability-vs-recoverability) I covered the difference between availability and recoverability. Fault isolation boundaries give us a way to quantify how we improve our availability. For the next and final post in this series, I'll cover the resilience spectrum and how we can effectively place our workloads on that spectrum.