
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

ref: [A word on Scalability](https://www.allthingsdistributed.com/2006/03/a_word_on_scalability.html)
ref: [Scalability, availability, stability, patterns](https://www.slideshare.net/slideshow/scalability-availability-stability-patterns/4062682)
