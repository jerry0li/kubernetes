
# System Design

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->

<!-- code_chunk_output -->

- [Scalability](#scalability)
    - [Clones](#clones)
    - [Databases](#databases)
    - [Caches](#caches)
    - [Asynchronism](#asynchronism)
- [High-level trade-offs](#high-level-trade-offs)
    - [Performance vs scalability](#performance-vs-scalability)
    - [Latency vs throughput](#latency-vs-throughput)
    - [Availability vs consistency](#availability-vs-consistency)
- [Domain name system](#domain-name-system)
    - [Disadvantages: DNS](#disadvantages-dns)

<!-- /code_chunk_output -->



## Scalability

#### Clones

Public servers of a scalable web service are hidden behind a load balancer. This load balancer evenly distributes load (requests from your users) onto your cluster of application servers. That means that if, for example, user Steve interacts with your service, he may be served at his first request by server 2, then with his second request by server 9 and then maybe again by server 2 on his third request.

Steve should always get the same results of his request back, independent what server he  “landed on”. That leads to the first golden rule for scalability: **every server contains exactly the same codebase and does not store any user-related data, like sessions or profile pictures, on local disc or memory**. 

**Sessions need to be stored in a centralized data store which is accessible to all your application servers**. It can be an external database or an external persistent cache, like Redis. **An external persistent cache will have better performance than an external database**. By external I mean that the data store does not reside on the application servers. Instead, it is somewhere in or near the data center of your application servers.


But what about deployment? **How can you make sure that a code change is sent to all your servers without one server still serving old code? This tricky problem is fortunately already solved by the great tool Capistrano**. It requires some learning, especially if you are not into Ruby on Rails, but it is definitely both the effort.


After “outsourcing” your sessions and serving the same codebase from all your servers, you can now create an image file from one of these servers (AWS calls this AMI - Amazon Machine Image.) Use this AMI as a “super-clone” that all your new instances are based upon. Whenever you start a new instance/clone, just do an initial deployment of your latest code and you are ready!


#### Databases

Now your servers can horizontally scale and you can already serve thousands of concurrent requests. But somewhere down the road your application gets slower and slower and finally breaks down. The reason: your database. It's something like MySQL, isn't it?

The required changes are more radical than just adding more cloned servers and may even require some boldness. In the end, you can choose from 2 paths:

**Path #1** is to stick with MySQL and keep the "beast" running. Hire a database administrator (DBA) tell him to do master-slave replication (read from slaves, write to master) and upgrade your master server by adding RAM, RAM and more RAM. In some months, your DBA will come up with words like "sharding", "denormalization" and "SQL tuning" and will look worried about the necessary overtime during the next weeks. At that point every new action to keep your database running will be more expensive and time consuming than the previous one. You might have been better off if you had chosen Path #2 while your database was still small and easy to migrate.

**Path #2** means to denormaliza right from the beginning and include no more Joins in any database query. You can stay with MySQL, and use it like a NoSQL database, or you can switch to a better and easier to scale NoSQL database like MongoDB or CouchDB. Joins will now need to be done in your application code. The sooner you do this step the less code you will have to change in the future. But even if you successfully switch to the latest and greatest NoSQL database and let your app do the dataset-joins, soon your database requests will again be slower and slower. You will need to introduce a cache.

#### Caches

Now you have a scalable database solution. You have no fear of storing terabytes anymore and the world is looking fine. But just for you. Your users still have to suffer slow page requests when a lot of data is fetched from the database. The solution is the implementation of a cache.

With "cache" I always mean in-memory caches like Redis or Memcached. **Please never do file-based caching**, it makes cloning and auto-scaling of your servers just a pain.

But back to in-memory caches. A cache is a simple key-value store and it should reside as a buffering layer between your application and your data storage. Whenever your application has to read data it should at first try to retrieve the data from your cache. Only if it's not in the cache should it then try to get the data from the main data source. Why should you do that? Because **a cache is lightning-fast**. It holds every dataset in RAM and requests are handled as fast as technically possible. For example, Redis can do several hundreds of thousands of read operations per second when being hosted on a standard server. Also writes, especially increments, are very, very fast. Try that with a database!

There are 2 patterns of caching your data. An old one and a new one:

**#1-Cached Database Queries**

That's still the most commonly used caching pattern. Whenever you do a query to your database, you store the result dataset in cache. A hashed version of your query is the cache key. The next time you run the query, you first check if it is already in the cache. The next time you run the query, you check at first the cache if there is already a result. This pattern has several issues. **The main issue is the expiration. It is hard to delete a cached result when you cache a complex query (who has not?). When one piece of data changes (for example a table cell), you need to delete all cached queries who may include that table cell**.

**#2-Cached Objects**

That's my strong recommendation and I always prefer this pattern. In general, see your data as an object like you already do in your code (classes, instances, etc.). Let your class assemble a dataset from your database and then store the complete instance of the class or the assembed dataset in the cache. Sounds theoretical, I know, but just look how you normally code. You have, for example, a class called "Product" which has a property called "data". It is an array containinng prices, texts, pictures, and customer reviews of your product. The property "data" is filled by several methods in the class doing several database requests which are hard to cache, since many things relate to each other. Now, do the following: when your class has finished the "assembling" of the data array, directly store the data array, or better yet the complete instance of the class, in the cache! This allows you to easily get rid of the object whenever something did change and makes the overall operation of your code faster and more logical.

And the best part: it makes asynchronous processing possible! Just imagine an army of worker servers who assemble your objects for you! The application just consumes the latest cached object and nearly never touches the databases anymore!


Some ideas of objects to cache:

+ user sessions (never use the database!)
+ fully rendered blog articles
+ activity streams
+ user <-> friend relationships


#### Asynchronism

Please imagine that you want to buy bread at your favorite bakery. So you go into the bakery, ask for a loaf of bread, but there is no bread there! Instead, you are asked to come back in 2 hours when your ordered bread is ready. That's annoying, isn't it?

To avoid such a "please wait a while" - situation, asynchronism needs to be done. And what's good for a bakery, is maybe also good for your web service or web app.

In general, there are two ways / paradigms asynchronism can be done.

**Async #1**
Let's stay in the former bakery picture. The first way of async processing is the "bake the breads at night and sell them in the morning" way. No waiting time at the cash register and a happy customer. Referring to a web app this means doing the time-consuming work in advance and serving the finished work with a low request time.

Very often this paradigm is used to turn dynamic content into static content. Pages of a website, maybe built with a massive framework or CMS, are prerendered and locally stored as static HTML files on every change. Often these computing tasks are done on a regular basis, maybe by a script which is called every hour by a cronjob. This pre-computing of overall general data can extremely improve websites and web apps and makes them very scalable and performant. Just imagine the scalability of your website if the script would upload these pre-rendered HTML pages to AWS S3 or Cloudfront or another Content Delivery Network! Your website would be super responsive and could handle millions of visitors per hour!

**Async #2**
Back to the bakery. Unfortunately, sometimes customers has special requests like a birthday cake with "Happy Birthday, Steve!" on it. The bakery can not foresee these kind of customer wishes, so it must start the task when the customer is in the bakery and tell him to come back at the next day. Refering to a web service that means to handle tasks asynchronously.

Here is a typical workflow:

A user comes to your website and starts a very computing intensive task which would take several minutes to finish. So the frontend of your website sends a job onto a job queue and immediately signals back to the user: your job is in work, please continue to the browse the page. The job queue is constantly checked by a bunch of workers for new jobs. If there is a new job then the worker does the job and after some minutes sends a signal that the job was done. The frontend, which constantly checks for new "job is done" - signals, sees that the job was done and informs the user about it. I know, that was a very simplified example.

If you now want to dive more into the details and actual technical design, I recommend you take a look at the first 3 tutorails on the [RabbitMQ website](https://web.archive.org/web/20230221220524/http://www.rabbitmq.com/). RabbitMQ is one of many systems which help to implement async processing. You could also use ActiveMQ or a simple [Redis list](https://web.archive.org/web/20230406150152/https://redis.io/docs/data-types/). The basic idea is to have a queue of tasks or jobs that a worker can process. Asynchronism seems complicated, but it is definitely worth your time to learn about it and implement it yourself. Backends become nearly infinitely scalable and frontends become snappy which is good for the overall user experience.

If you do something time-consuming, try to do it always asynchronously.


## High-level trade-offs

Keep in mind that **everything is a trade-off.**

#### Performance vs scalability

A service is scalable if it results in increased performance in a manner proportional to resources added. Generally, increasing performance means serving more units of work, but it can also be to handle larger units of work, such as when datasets grow.

Another way to look at performance vs scalability:

+ If you have a performance problem, your system is slow for a single user.
+ If you have a scalability problem, your system is fast for a single user but slow under heavy load.


Scalability is frequently used as a magic incantation to indicate that something is badly designed or broken. Often you hear in a discussion "but that doesn't scale" as the magical word to end an argument. This is often an indication that developers are running into situations where the architecture of their system limits their ability to grow their service. If scalability is used in a positive sense it is in general to indicate a desired property as in "our platform needs good scalability".

What is it that we really mean by scalability? *A service is said to be scalable if when we increase the resources in a system, it results in increased performance in a manner proportional to resources added*. Increasing performance in general means serving more units of work, but it can also be to handle larger units of work, such as when datasets grow.

In distributed systems there are other reasons for adding resources to a system; for example to improve the reliability of the offered service. Introducing redundancy is an important first line of defense against failures. *An always-on service is said to be scalable if adding resources to facilitate redundancy does not result in a loss of performance*.


Why is scalability so hard? Because scalability cannot be an after-thought. It requires applications and platforms to be designed with scaling in mind, such that adding resources actually results in improving the performance or that if redundancy is introduced the system performance is not adversely affected. Many algorithms that perform reasonably well under low load and small datasets can explode in cost if either requests rates increase, the dataset grows or the number of nodes in the distributed system increases.

A second problem area is that growing a system throught scale-out generally results in a system that has to come to terms with heterogeneity. Resources in the system increase in diversity as next generations of hardware come on line, as bigger or more powerful resources become more cost-effective or when some resources are placed further apart. Heterogeneity means that some nodes will be able to process faster or store more data than other nodes in a system and algorithms that rely on uniformity either break down under these conditions or underutilize the newer resources.

Is achieving good scalability possible? Absolutely, but only if we architect and engineer our systems to take scalability into account. For the systems we build we must carefully inspect along which axis we expect the system to grow, where redundancy is required, and how one should handle heterogeneity in this system, and make sure that architects are aware of which tools they can use for under which conditions, and what the common pitfalls are.

ref: [A word on Scalability](https://www.allthingsdistributed.com/2006/03/a_word_on_scalability.html)
ref: [Scalability, availability, stability, patterns](https://www.slideshare.net/slideshow/scalability-availability-stability-patterns/4062682)


#### Latency vs throughput

**Latency** is the time to perform some action or to product some result.

**Throughput** is the number of such actions or results per unit of time.

Generally, you should aim for **maximal throughput** with **acceptable latency**.


One of the effects of adopting a [High Level Synthesis design methodology](https://www.cadence.com/en_US/home/resources/white-papers/how-the-productivity-advantages-of-high-level-synthesis-can-improve-ip-design-verification-and-reuse-wp.html) is that the barrier between "Systems designers" and "Hardware designers" is substantially reduced if not totally eliminated. Suddenly, both "Systems designers" and "Hardware designers" are using not only the same input language to specify their models (C++) but they are also exposed to the same terminology. For this reason, "Hardware designers" are suddenly exposed to two terms to wchi they have had little or no exposure in the past.

The purpose of this post is to clarify two "systems" terms that are usually confused and sometimes used interchangeably: latency and throughput.

**Definition of terms**

Let us attempt to define those two terms:

**Latency** is the time required to perform some action or to produce some result. Latency is measured in units of time -- hours, minutes, seconds, nanoseconds or clock periods.

**Throughput** is the number of such actions executed or results produced per unit of time. This is measured in units of whatever is being produced (cars, motorcycles, I/O samples, memory words, iterations) per unit of time. The term "memory bandwidth" is sometimes used to specify the throughput of memory systems.

**A simple example**

The following manufacturing example should clarify these two concepts:

An assembly line is manufacturing cars. It takes eight hours to manufacture a car and that the factory produces one hundred and twenty cars per day.

The latency is: 8 hours.

The throughput is: 120 cars/day or 5 cars/hour.


#### Availability vs consistency

**CAP theorem**

In a distributed computer system, you can only support two of the following guarantees:

*Consistency* - Every read receives the most recent write or an error

*Availability* - Every request receives a response, without guarantee that it contains the most recent version of the information

*Parttion Tolerance* - The system continues to operate despite arbitrary partitioning due to network failures.

Networks aren't reliable, so you'll need to support partition tolerance. You'll need to make a software tradeoff between consistency and availability.

**CP** - consistency and partition tolerance

Waiting for a response from the partitioned node might result in a timeout error. CP is a good choice if your business needs require atomic reads and writes.

**AP** - availability and partition tolerance

Response return the most readily available version of the data available on any node, which might not be the latest. Writes might take some time to propagate when the partition is resolved.

AP is a good choice if the business needs to allow for eventual consistency or when the system needs to continue working despite external errors.

ref: [CAP theorem revisited](https://robertgreiner.com/cap-theorem-revisited/)
ref: [A plain english introduction to CAP theorem](http://ksat.me/a-plain-english-introduction-to-cap-theorem)
ref: [CAP FAQ](https://github.com/henryr/cap-faq)
ref: [The CAP theorem](https://www.youtube.com/watch?v=k-Yaq8AHlFA)


##### Consistency patterns

With multiple copies of the same data, we are faced with options on how to synchronize them so clients have a consistent view of the data. Recall the definition of consistency from the CAP theorem - Every read receives the most recent write or an error.

**Weak consistency**

After a while, reads may or may not see it. A best effort approach is taken.

This approach is seen in systems such as memcached. Weak consistency works well in real time use cases such as VoIP, video chat, and realtime multiplayer games. For example, if you are on a phone call and lose reception for a few seconds, when you regain connection you do not hear what was spoken during connection loss.


**Eventual consistency**

After a write, reads will eventually see it (typically within milliseconds). Data s replicated asynchronously.

This approach is seen in systems such as DNS and email. Eventual consistency works well in highly available systems.


**Strong consistency**

After a while, reads will see it. Data is replicated synchronously.

This approach is seen in file systems and RDBMSes. Strong consistency works well in systems that need transactions.

Ref: [Transactions across data centers](https://snarfed.org/transactions_across_datacenters_io.html)


##### Availability patterns

There are two complementary patterns to support high availability: fail-over and replication

**Fail-over**

+ Active-passive: With active-passive fail-over, heartbeats are sent between the active and the passive server on standby. If the heartbeat is interrupted, the passive server takes over the active's IP address and resumes service.

  The length of downtime is determined by whether the passive server is already running in 'hot' standby or whether it needs to start up from 'cold' standby. Only the active server handles traffic.

  Active-passive failover can also be referred to as master-slave failover.


+ Active-active: In active-active, both servers are managing traffic, spreading the load between them.

  If the servers are public-facing, the DNS would need to know about the public IPs of both servers. If the servers are internal-facing, application logic would need to know about both servers.

  Active-active failover can also be referred to as master-master failover.

**Disadvantages: failover**

+ Fail-over adds more hardware and additional complexity.
+ There is a potential for loss of data if the active system fails before any newly written data can be replicated to the passive.
  
##### Replication

**Master-slave and master-master**

This topic is further discussed in the Database section

**Availability in numbers**

Availability is often quantified by uptime (or downtime) as a percentage of time the service is available. Availability is generally measured in number of 9s--a service with 99.99% availability is described as having four 9s.

**99.9 availability - thres 9s**

| Duration | Acceptable downtime | 
| -- | -- |
| Downtime per year | 8h 45min 57s| 
| Downtime per month | 43m 49.7s |
| Downtime per week | 10m 4.8s |
| Downtime per day | 1m 26.4s |

**99.99% availability - four 9s**

| Duration | Acceptable downtime | 
| -- | -- |
| Downtime per year | 52min 35.7s | 
| Downtime per month | 4m 23s | 
| Downtime per week | 1m 5s |
| Downtime per day | 8.6s |

**Availability in parallel vs in sequence**

If a service consists of multiple components prone to failure, the service's overall availability depends on whether the components are in sequence or in parallel.

+ In sequence: Overall availability decreases when two components with availability < 100% are in sequence:
  ```
  Availability (Total) = Availability (Foo) * Availability (Bar)
  ```

  If both `Foo` and `Bar` each had 99.9% availability, their total availability in sequence would be 99.8%.

+ In parallel: Overall availability increases when two components with availability < 100% are in parallel:
  ```
  Availability (Total) = 1 - (1 - Availability (Foo)) * (1 - Availability (Bar))
  ```

  If both `Foo` and `Bar` each had 99.9% availability, their total availability in parallel would be 99.9999%.


## Domain name system

How does DNS work?

![](../imgs/dns.png)

A Domain Name System (DNS) translates a domain name such as www.example.com to an IP address.

DNS is hierarchical, with a few authoritative servers at the top level. Your router or ISP provides information about which DNS server(s) to contact when doing a lookup. Lower level DNS servers cache mappings, which could become stale due to DNS propagation delays. DNS results can also be cached by your browser or OS for a certain period of time, determined by [time to live](https://en.wikipedia.org/wiki/Time_to_live)

+ NS record (name server) - Specifies the DNS servers for your domain/subdomain.

+ MX record (mail exchange) - Specifies the mail servers for accepting messages.

+ A record (address) - Points a name to an IP address.

+ CNAME (canonical) - Points a name to another name or `CNAME` or to an `A` record


Services such as [CloudFlare](https://www.cloudflare.com/dns/) and [Route 53](https://aws.amazon.com/route53/) provide managed DNS services. Some DNS services can route traffic through various methods:

+ [Weighted round robin](https://www.jscape.com/blog/load-balancing-algorithms)
  - Prevent traffic from going to servers under maintenance
  - Balance between varying cluster sizes
  - A/B testing

+ [Latency-based](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html)
  
+ [Geolocation-based](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)


#### Disadvantages: DNS

+ Accessing a DNS server introduces a slight delay, although mitigated by caching described above.
+ DNS server management could be complex and is generally managed by [governments, ISPs, and large companies.](http://superuser.com/questions/472695/who-controls-the-dns-servers/472729)
+ DNS servicess have recently come under [DDoS attack](http://dyn.com/blog/dyn-analysis-summary-of-friday-october-21-attack/), preventing users from accessing websites such as Twitter without knowing Twitter's IP address.