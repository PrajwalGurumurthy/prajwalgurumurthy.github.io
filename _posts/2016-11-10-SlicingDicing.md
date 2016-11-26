---
layout: post
title: Gluing the Sliced/Diced Metadata in a Distributed way
description: To slice and dice is to break a body of information down into smaller parts. Gluing them back using an active-active deployment for scalability and availability is quiet a challenging task.
---
> Problem Description

The Metadata Gluer is responsible of Glueing metadata of content/contents from multiple sources. The content metadata can arrive in many pieces and the metadata aggregator is responsible of glueing these pieces. For example lets say we have a content C1 in XML/Json, and its metadata arrives in many pieces C1P1(content1Piece1),C1P2 ... over a discrete interval of time(not necessarily continious).For simplicity lets assume there is a queue for queung all these piece by piece updates 'Cnpn'.And each of the piece can insert/update/delete different parts of the content XML/JSON.Note that the final metadata of content must be consistent. Also note that as and when pieces are glued, the current state of content is sent as an update for downstream components which process this update to the content.

![Image1]({{ site.url }}/assets/sd/sd1.png)

> Challenge

The challenge here is to build an active-active solution where there are multiple instances of Metadata Gluer running concurrently.This is extremely important since the data load can be too high specially in a multi tenant deployment and the granularity of the pieces are small and frequent.

>Solution1: Active-Passive

Before we get into the active-active solution, lets try to understand the  Active-Passive solution.There is one Metadata gluer instance which processes each pieces one after the other for all contents and glues each piece update one after the other and notifies the downstream components. If the active instance goes down there is a passive instance which can takeover from active and continue.

![Image1]({{ site.url }}/assets/sd/sd2.png)

>Pros and Cons
<b>Pros</b>

{% highlight text %}
1.  Easy Deployment Model
2.  Very Simple implementation.All the Gluer has to do is join pieces as they arrive.
3.  Suitabe when the updates are infrequent and data load is less.
{% endhighlight %}

<b>Cons</b>

{% highlight text %}
1.  Does not scale with data load.
2.  Multi tenancy will require separate instances of gluer for each tenant. Even then it might not scale with higher data load per tenant.
3.  Costlier deployment if the number of tenants are more.
4.  Higher Maintainance iwth large number of deployments
5.  Not suitable for SAAS kind of model
6.  The granular updates (C1P1...CnPn) are queued for long time since there is only a single instance processing these updates.Hence downstream components receive the updates very slowly.As a result takes larger time for update propogation downstream.

{% endhighlight %}

That brings us to the active active solution which addresses all these concerns.

>Solution2: Active-Active (Distributed Glueing)

There are multiple instances of metadata gluer processing the updates concurrently.As and when the updates arrive they are picked up by the free active instance and starts processing.

![Image1]({{ site.url }}/assets/sd/sd3.png)

Just adding multiple instances will not solve the problem since there is a constraint that the updates(C1P1,C1P2,C1P3...C1Pn) of a  particular content (C1) has to be processed in order and the final state of content (C1) has to be consistent. The above architecture will not address this constraint since there is a possibilitily that updates C1P1,C1P2...C1Pn can be adjacent in the queue and multiple gluer instances can take these updates simultaneously.As a result the final state of the content (C1) might be inconsistent
<br>
To address this challenge lets tweak the above architecture and make it flexible enough to address the above issue.Lets have a separate queue for each metadata gluer instance as shown below.

![Image1]({{ site.url }}/assets/sd/sd4.png)

The queue creation and events forwarding to each queue is taken care by many message bus technologies like RMQ,KAFKA. So the queue management responsibility is offloaded to Message bus.
<br>

>Here comes the Hash trick

Many distributed persistent technologies like cassandra,couchbase,mongo rely on hash based partitioning for data distribution across nodes.The same technique can be used here for load distribution across gluer instances.Each node is responsible for glueing the content for a range of hash values.

![Image1]({{ site.url }}/assets/sd/sd5.png)

For instance in the above example lets say gluer1 has a hash range of 1-30,gluer2 has 31-60 and gluer3 has 61-99. Since all of them have a mirror of update events, each of the gluer filters the events based on the above hash range. Gluer1 only process contents 1-30 and rejects all events that are outside its range. Therefore the Gluer1 process the updates(C1P1,C1P2,C1P3...C1Pn) in order there by ensuring the consistentcy of the contents.
Note that the load is shared among the nodes yet ensuring the consistency.
<br>
<b>Gluer1 Flow of events</b>
<br>

{% highlight text %}

C1P1    : C1 is in the hash range, processes the C1P1 event and glues the update in C1.
C20P1   : C20 is in the hash range, processes the C20P1 event and glues the update in C20.
C1P2    : C1 is in the hash range, processes the C1P2 event and glues the update in C1.
C1P3    : C1 is in the hash range, processes the C1P3 event and glues the update in C1.

[Note the updates C1P3 is applied only after C1P2 making sure that C1 is in consitent state]

C20P2   : C20 is in the hash range, processes the C20P2 event and glues the update in C20.
C40P1   : C40 is in not the hash range, Drops this event.[This will be processed by any one of the other gluer instances]
C30P2   : C30 is in the hash range, processes the C30P2 event and glues the update in C30.
C30P3   : C30 is in the hash range, processes the C30P3 event and glues the update in C30.
C1P4    : C1 is in the hash range, processes the C1P4 event and glues the update in C1.
C20P3   : C20 is in the hash range, processes the C20P3 event and glues the update in C20.

{% endhighlight %}

Similarly all the other instances filters the events based on their hash range and process the events.

So far so good. But still it doesn't fly. We need to figure out the following.

{% highlight text %}
How can the nodes determine the hashrange themselves? 
How can nodes update their hashrange when new nodes are added/deleted?
How do we handle the failure of nodes?
How do we scale based on load?
{% endhighlight %}

>How can the nodes determine the hashrange themselves?

>How can nodes update their hashrange when new nodes are added/deleted?

>How do we handle the failure of nodes?

>How do we scale based on load?

>Pros and Cons

<b>Pros</b>
{% highlight text %}
1.  Handles large data loads
2.  Suitable for multi tenant deployment
3.  Updates are propogated to downstream components faster.
4.  No separate deployment needed for each tenant
5.  Suitable for SAAS kind of model
{% endhighlight %}

<b>Cons</b>
{% highlight text %}
1.  Additional infra like messsage bus is needed.
2.  Complexity in handling aspects like node discovery,hash range distribution, failure handling.
{% endhighlight %}


We analyzed the pros and cons of Active/Passive and Active/Active approach of solving the gluing of content metadata.The Active/Passive does solve the problem in a simple way but it clearly doesn't scale with load.If there is a need to support multitenant where the data load will be higher active/active approach is the way. But it comes with its own baggage. The decision clearly depends on the requirement.
