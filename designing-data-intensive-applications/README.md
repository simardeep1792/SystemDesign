# Designing Data Intensive Applications

## Chapter 1 - Reliable, Scalable, and Maintainable Applications

* Many applications today are data-intensive, as opposed to compute-intensive. Raw CPU power is rarely a limiting factor for these applications—bigger problems are usually the amount of data, the complexity of data, and the speed at which it is changing.

* A data-intensive application is typically built from standard building blocks that provide commonly needed functionality. For example, many applications need to:
    * Store data so that they, or another application, can find it again later (databases)
    * Remember the result of an expensive operation, to speed up reads (caches)
    * Allow users to search data by keyword or filter it in various ways (search indexes)
    * Send a message to another process, to be handled asynchronously (stream processing)
    * Periodically crunch a large amount of accumulated data (batch processing)


* ![overview](./images/overview.png)

* Reliability
    * The system should continue to work correctly (performing the correct function at the desired level of performance) even in the face of adversity (hardware or software faults, and even human error). See “Reliability” on page 6.
* Scalability
    * As the system grows (in data volume, traffic volume, or complexity), there should be reasonable ways of dealing with that growth. See “Scalability” on page 10.
* Maintainability
    * Over time, many different people will work on the system (engineering and operations, both maintaining current behavior and adapting the system to new use cases), and they should all be able to work on it productively. See “Maintainability” on page 18.

* Hardware Faults
    * Hard disks are reported as having a mean time to failure (MTTF) of about 10 to 50 years [5, 6]. Thus, on a storage cluster with 10,000 disks, we should expect on average one disk to die per day.

* Software Errors
    * A software bug that causes every instance of an application server to crash when given a particular bad input. For example, consider the leap second on June 30, 2012, that caused many applications to hang simultaneously due to a bug in the Linux kernel [9].
    * A runaway process that uses up some shared resource—CPU time, memory, disk space, or network bandwidth.
    * A service that the system depends on that slows down, becomes unresponsive, or starts returning corrupted responses.
    * Cascading failures, where a small fault in one component triggers a fault in another component, which in turn triggers further faults [10].

* Human Errors
    * Humans design and build software systems, and the operators who keep the systems running are also human. Even when they have the best intentions, humans are known to be unreliable. For example, one study of large internet services found that configuration errors by operators were the leading cause of outages, whereas hardware faults (servers or network) played a role in only 10–25% of outages [13].

* Describing Load
    * First, we need to succinctly describe the current load on the system; only then can we discuss growth questions (what happens if our load doubles?). Load can be described with a few numbers which we call load parameters. The best choice of parameters depends on the architecture of your system: it may be requests per second to a web server, the ratio of reads to writes in a database, the number of simultaneously active users in a chat room, the hit rate on a cache, or something else. Perhaps the average case is what matters for you, or perhaps your bottleneck is dominated by a small number of extreme cases.

    * Twitter usecase
        * Post a tweet
            * A user can publish a new message to their followers (`4.6k requests/sec` on average, over `12k requests/sec` at peak).
        * Home timeline
            * A user can view tweets posted by the people they follow (`300k requests/sec`).

        * Simply handling `12,000 writes per second` (the peak rate for posting tweets) would be fairly easy. However, Twitter’s scaling challenge is not primarily due to `tweet volume`, but due to **fan-out** — each user follows many people, and each user is followed by many people. There are broadly two ways of implementing these two operations:
            * `fan-out`: In transaction processing systems, we use it to describe the number of requests to other services that we need to make in order to serve one incoming request.
        
        * Posting a tweet simply inserts the new tweet into a global collection of tweets. When a user requests their home timeline, look up all the people they follow, find all the tweets for each of those users, and merge them (sorted by time). In a relational database like in Figure 1-2, you could write a query such as:

        * ![twitter_usecase](./images/twitter_usecase.png)  

        * **Maintain a cache for each user’s home timeline** like a mailbox of tweets for each recipient user (see Figure 1-3). When a user posts a tweet, look up all the people who follow that user, and insert the new tweet into each of their home timeline caches. The request to read the home timeline is then cheap, because its result has been computed ahead of time.

        * ![twitter_mailbox](./images/twitter_mailbox.png)  

        * The first version of `Twitter` used `approach 1`, but the systems struggled to keep up with the load of home timeline queries, so the company switched to `approach 2`. This works better because **the average rate of published tweets is almost two orders of magnitude lower than the rate of home timeline reads**, and so in this case it’s preferable **to do more work at write time and less at read time**.

        * However, the downside of `approach 2` is that posting a tweet now requires a lot of extra work. On average, a tweet is delivered to about `75 followers`, so `4.6k tweets per second become 345k` writes per second to the home timeline caches. But this average hides the fact that the **number of followers per user varies wildly**, and some users have over `30 million followers`. This means that a single tweet may result in over `30 million` writes to home timelines! Doing this in a timely manner—Twitter tries to deliver tweets to followers within five seconds — is a significant challenge.

        * The final twist of the `Twitter` anecdote: now that `approach 2` is robustly implemented, Twitter is moving to a `hybrid` of both approaches. Most users’ tweets continue to be fanned out to home timelines at the time when they are posted, but a small number of users with a very large number of followers (i.e., celebrities) are excepted from this `fan-out`.

* Describing Performance
    * Once you have described the load on your system, you can investigate what happens when the load increases. You can look at it in two ways:

        * When you increase a load parameter and keep the system resources (CPU, memory, network bandwidth, etc.) unchanged, **how is the performance of your system affected**?
        * When you increase a load parameter, how much do you need to **increase the resources** if you want to keep performance unchanged?
    
    * In a batch processing system such as Hadoop, we usually care about **throughput the number of records we can process per second**, or the total time it takes to run a job on a dataset of a certain size.iii In online systems, what’s usually more important is the service’s **response time—that** is, the time between a client sending a request and receiving a response.
        * iii. In an ideal world, the running time of a batch job is the size of the dataset divided by the throughput. In practice, the running time is often longer, due to skew (data not being spread evenly across worker processes) and needing to wait for the slowest task to complete.
    
    * **Latency and response time are often used synonymously, but they are not the same.** The response time is what the client sees: besides the actual time to process the request (the service time), it includes network delays and queueing delays. Latency is the duration that a request is `waiting to be handled` during which it is latent, await‐ ing service.

    * Even if you only make the same request over and over again, you’ll get a slightly dif‐ ferent response time on every try. **In practice, in a system handling a variety of requests, the response time can vary a lot. We therefore need to think of response time not as a single number, but as a distribution of values that you can measure.**

    * ![response_time](./images/response_time.png)
        * The median is the middle number in a sorted, ascending or descending, list of numbers and can be more descriptive of that data set than the average. The median is sometimes used as opposed to the mean when there are outliers in the sequence that might skew the average of the values.
    
    * It’s common to see the average response time of a service reported. (Strictly speaking, the term “average” doesn’t refer to any particular formula, but in practice it is usually understood as the arithmetic mean: given n values, add up all the values, and divide by n.) **However, the mean is not a very good metric if you want to know your “typical” response time, because it doesn’t tell you how many users actually experienced that delay.**

    * **Usually it is better to use percentiles. If you take your list of response times and sort it from fastest to slowest, then the median is the halfway point: for example, if your median response time is 200 ms, that means half your requests return in less than 200 ms, and half your requests take longer than that.**

    * The **median** is also known as the 50th percentile, and sometimes abbreviated as **p50**. 

    * In order to figure out how bad your outliers are, you can look at higher percentiles: the `95th`, `99th`, and `99.9th` percentiles are common (abbreviated `p95`, `p99`, and `p999`). They are the response time thresholds at which 95%, 99%, or 99.9% of requests are faster than that particular threshold. For example, if the `95th` percentile response time is 1.5 seconds, that means `95 out of 100 requests take less than 1.5 seconds`, and 5 out of 100 requests take 1.5 seconds or more.

    * High percentiles of response times, also known as tail latencies, are important because they directly affect users’ experience of the service. For example, Amazon describes response time requirements for internal services in terms of the 99.9th per‐ centile, even though it only affects 1 in 1,000 requests. This is because the customers with the slowest requests are often those who have the most data on their accounts because they have made many purchases.
        * That is, they’re the most valuable customers [19]. It’s important to keep those customers happy by ensuring the website is fast for them: Amazon has also observed that a 100 ms increase in response time reduces sales by 1% [20], and others report that a 1-second slowdown reduces a customer sat‐ isfaction metric by 16%
    
    * For example, percentiles are often used in `service level objectives (SLOs)` and `service level agreements (SLAs)`, contracts that define the expected performance and availability of a service. An `SLA` may state that the service is considered to be up if it has a `median response time of less than 200 ms` and a `99th percentile under 1 s` (if the response time is longer, it might as well be down), and the service may be required to be up at least 99.9% of the time. These metrics set expectations for clients of the ser‐ vice and allow customers to demand a refund if the SLA is not met.

    * **Queueing delays** often account for a large part of the `response time` at `high percentiles`. As a server can only process a `small number of things in parallel` (limited, for example, by its **number of CPU cores**), **it only takes a small number of slow requests to hold up the processing of subsequent requests an effect** sometimes known as `head-of-line blocking`. Even if those subsequent requests are fast to process on the server, the client will see a slow overall response time due to the time waiting for the prior request to complete. Due to this effect, it is important to measure response times on the client side.

    * ![tail_latency](./images/tail_latency.png)

* Approaches for Coping with Load
    * how do we maintain good performance even when our load parameters increase by some amount?

    * People often talk of a dichotomy between `scaling up` (vertical scaling, moving to a more powerful machine) and `scaling out` (horizontal scaling, distributing the load across multiple smaller machines). Distributing load across multiple machines is also known as a shared-nothing architecture. A system that can run on a single machine is often simpler, but high-end machines can become very expensive, so very intensive workloads often can’t avoid scaling out. In reality, good architectures usually involve a pragmatic mixture of approaches: for example, using several fairly powerful machines can still be simpler and cheaper than a large number of small virtual machines.

    * While distributing stateless services across multiple machines is fairly straightforward, taking stateful data systems from a single node to a distributed setup can intro‐ duce a lot of additional complexity. For this reason, common wisdom until recently was to keep your database on a single node (scale up) until scaling cost or high- availability requirements forced you to make it distributed.

    * **As the tools and abstractions for distributed systems get better, this common wisdom may change, at least for some kinds of applications. It is conceivable that distributed data systems will become the default in the future, even for use cases that don’t handle large volumes of data or traffic.** 

    * The architecture of systems that operate at large scale is usually **highly specific to the application there is no such thing as a generic, one-size-fits-all scalable architecture** (informally known as magic scaling sauce). The problem may be the `volume of reads`, the `volume of writes`, the `volume of data to store`, the `complexity of the data`, the `response time requirements`, the `access patterns`, or (usually) some mixture of all of these plus many more issues.

* Maintainability
    * It is well known that the **majority of the cost of software is not in its initial development, but in its ongoing maintenance—fixing bugs**, keeping its systems operational, investigating failures, adapting it to new platforms, modifying it for new use cases, repaying technical debt, and adding new features.

    * Operability
        * Make it easy for operations teams to keep the system running smoothly.
    * Simplicity
        * Make it easy for new engineers to understand the system, by removing as much complexity as possible from the system. (Note this is not the same as simplicity of the user interface.)
    * Evolvability
        * Make it easy for engineers to make changes to the system in the future, adapting it for unanticipated use cases as requirements change. Also known as **extensibility, modifiability, or plasticity.**

* Operability: Making Life Easy for Operations

    * Operations teams are vital to keeping a software system running smoothly. A good operations team typically is responsible for the following, and more [29]:

        * Monitoring the health of the system and quickly **restoring service** if it goes into a bad state
        * **Tracking down the cause of problems**, such as system failures or degraded performance
        * Keeping software and **platforms up to date**, including security patches
        * Keeping tabs on **how different systems affect each other**, so that a problematic change can be avoided before it causes damage
        * **Anticipating future problems** and solving them before they occur (e.g., **capacity planning**)
        * Establishing good practices and **tools for deployment**, configuration management, and more
        * Performing complex maintenance tasks, such as **moving an application from one platform to another**
        * Maintaining the **security of the system** as configuration changes are made
        * Defining processes that make operations predictable and help keep the production environment stable
        * Preserving the organization’s knowledge about the system, even as individual people come and go
    
    * Data systems can do various things to make routine tasks easy, including:
        * Providing **visibility into the runtime behavior** and internals of the system, with good monitoring
        * Providing **good support for automation and integration** with standard tools
        * **Avoiding dependency on individual machines** (allowing machines to be taken down for maintenance while the system as a whole continues running uninterrupted)
        * Providing **good documentation** and an easy-to-understand operational model (“If I do X, Y will happen”)
        * Providing **good default behavior**, but also giving administrators the freedom to override defaults when needed
        * **Self-healing where appropriate**, but also giving administrators manual control over the system state when needed
        * Exhibiting **predictable behavior**, minimizing surprises

* Simplicity: Managing Complexity
    * Small software projects can have delightfully simple and expressive code, but as projects get larger, they often become very complex and difficult to understand. This complexity slows down everyone who needs to work on the system, further increasing the cost of maintenance. A software project mired in complexity is sometimes described as a big ball of mud [30].

    * There are various possible symptoms of complexity: explosion of the state space, tight coupling of modules, tangled dependencies, inconsistent naming and terminology, hacks aimed at solving performance problems, special-casing to work around issues elsewhere, and many more. Much has been said on this topic already [31, 32, 33].

    * **One of the best tools we have for removing accidental complexity is abstraction. A good abstraction can hide a great deal of implementation detail behind a clean, simple-to-understand façade.**
        * For example, high-level programming languages are abstractions that hide machine code, CPU registers, and syscalls. SQL is an abstraction that hides complex on-disk and in-memory data structures, concurrent requests from other clients, and inconsistencies after crashes.

* Evolvability: Making Change Easy
    * It’s extremely unlikely that your system’s requirements will remain unchanged for‐ ever. They are much more likely to be in constant flux: you learn new facts, previously unanticipated use cases emerge, business priorities change, users request new features, new platforms replace old platforms, legal or regulatory requirements change, growth of the system forces architectural changes, etc.

    * **The Agile community has also developed technical tools and pat‐ terns that are helpful when developing software in a frequently changing environment, such as test-driven development (TDD) and refactoring.**

## Chapter 2 - Data Models and Query Languages

* Data models are perhaps the most important part of developing software, because they have such a profound effect: not only on how the software is written, but also on how we think about the problem that we are solving.

* In this chapter we will look at a range of general-purpose data models for data stor‐ age and querying (point 2 in the preceding list). In particular, we will compare the relational model, the document model, and a few graph-based data models. We will also look at various query languages and compare their use cases. In Chapter 3 we will discuss how storage engines work; that is, how these data models are actually implemented (point 3 in the list).

### Relational Model Versus Document Model
* The best-known data model today is probably that of SQL, based on the relational model proposed by Edgar Codd in 1970 [1]: data is organized into relations (called tables in SQL), where each relation is an unordered collection of tuples (rows in SQL).

* The roots of relational databases lie in business data processing, which was performed on mainframe computers in the 1960s and ’70s. The use cases appear mundane from today’s perspective: typically transaction processing (entering sales or banking trans‐ actions, airline reservations, stock-keeping in warehouses) and batch processing (cus‐ tomer invoicing, payroll, reporting).

* **Over the years, there have been many competing approaches to data storage and querying. In the 1970s and early 1980s, the `network model` and the `hierarchical model` were the main alternatives, but the relational model came to dominate them.** Object databases came and went again in the late 1980s and early 1990s. XML databases appeared in the early 2000s, but have only seen niche adoption. Each competitor to the relational model generated a lot of hype in its time, but it never lasted [2].

### The Birth of NoSQL

* Now, in the 2010s, NoSQL is the latest attempt to overthrow the relational model’s dominance. The name “NoSQL” is unfortunate, since it doesn’t actually refer to any particular technology—it was originally intended simply as a catchy Twitter hashtag for a meetup on open source, distributed, nonrelational databases in 2009.

* There are several driving forces behind the adoption of NoSQL databases, including:

    * A need for **greater scalability than relational databases** can easily achieve, including very large datasets or **very high write throughput**
    * A widespread preference for free and open source software over commercial database products
    * Specialized query operations that are not well supported by the relational model
    * Frustration with the restrictiveness of relational schemas, and a desire for a more dynamic and expressive data model [5]

* Different applications have different requirements, and the best choice of technology for one use case may well be different from the best choice for another use case. It therefore seems likely that in the foreseeable future, relational databases will continue to be used alongside a broad variety of nonrelational datastores—an idea that is sometimes called **polyglot persistence** [3].

### The Object-Relational Mismatch

* Most application development today is done in object-oriented programming languages, which leads to a common criticism of the SQL data model: if data is stored in relational tables, an `awkward translation layer` is required between the objects in the application code and the database model of tables, rows, and columns. The disconnect between the models is sometimes called an **impedance mismatch**.

    * `Object-relational mapping (ORM)` frameworks like ActiveRecord and Hibernate reduce the amount of boilerplate code required for this translation layer, but they can’t completely hide the differences between the two models.

* ![cv](./images/cv.png) 

* In the traditional SQL model (prior to SQL:1999), the `most common normalized representation` is to put positions, education, and contact information in separate tables, with a foreign key reference to the users table, as in Figure 2-1.

* Later versions of the SQL standard added support for `structured datatypes and XML data`; this allowed multi-valued data to be stored within a single row, with support for querying and indexing inside those documents. These features are supported to varying degrees by `Oracle, IBM DB2, MS SQL Server, and PostgreSQL` [6, 7]. A **JSON datatype is also supported by several databases**, including IBM DB2, MySQL, and PostgreSQL [8].

* A third option is to encode jobs, education, and contact info as a `JSON or XML` document, store it on a text column in the database, and let the application inter‐ pret its structure and content. In this setup, you typically cannot use the database to query for values inside that encoded column.

* For a data structure like a résumé, which is mostly a self-contained document, a JSON representation can be quite appropriate: see Example 2-1. `JSON` has the appeal of being much simpler than `XML`. **Document-oriented databases like MongoDB [9], RethinkDB [10], CouchDB [11], and Espresso [12] support this data model**.
    * Some developers feel that the `JSON` model reduces the **impedance mismatch** between the application code and the storage layer. However, as we shall see in Chapter 4, there are also problems with JSON as a data encoding format. The lack of a schema is often cited as an advantage; we will discuss this in “Schema flexibility in the document model” on page 39.
    * The `JSON` representation has **better locality than the multi-table schema** in Figure 2-1. If you want to fetch a profile in the relational example, you need to either perform multiple queries (query each table by user_id) or perform a messy multi- way join between the users table and its subordinate tables. In the **JSON representation, all the relevant information is in one place, and one query is sufficient.**

### Many-to-One and Many-to-Many Relationships

* If the user interface has free-text fields for entering the region and the industry, it makes sense to store them as plain-text strings. But there are advantages to having standardized lists of geographic regions and industries, and letting users choose from a drop-down list or autocompleter:
    * Consistent style and spelling across profiles
    * **Avoiding ambiguity** (e.g., if there are several cities with the same name)
    * **Ease of updating** —the name is stored in only one place, so it is easy to update across the board if it ever needs to be changed (e.g., change of a city name due to political events)
    * **Localization support** —when the site is translated into other languages, the standardized lists can be localized, so the region and industry can be displayed in the viewer’s language
    * **Better search** —e.g., a search for philanthropists in the state of Washington can match this profile, because the list of regions can encode the fact that Seattle is in Washington (which is not apparent from the string "Greater Seattle Area")

* The advantage of using an ID is that because it has no meaning to humans, it never needs to change: the ID can remain the same, even if the information it identifies changes. Anything that is meaningful to humans may need to change sometime in the future—and if that information is duplicated, all the redundant copies need to be updated. That incurs write overheads, and risks inconsistencies (where some copies of the information are updated but others aren’t).**Removing such duplication is the key idea behind normalization in databases.ii**

* **If the database itself does not support joins, you have to emulate a join in application code by making multiple queries to the database.** (In this case, the lists of regions and industries are probably small and slow-changing enough that the application can simply keep them in memory. But nevertheless, the work of making the join is shifted from the database to the application code.)

### Are Document Databases Repeating History?

* While many-to-many relationships and joins are routinely used in relational data‐ bases, document databases and NoSQL reopened the debate on how best to represent such relationships in a database. **This debate is much older than NoSQL—in fact, it goes back to the very earliest computerized database systems.

* The most popular database for business data processing in the 1970s was IBM’s `Information Management System (IMS)`, originally developed for `stock-keeping (SKU)` in the Apollo space program and first `commercially released in 1968` [13]. It is still in use and maintained today, `running on OS/390 on IBM mainframes` [14].
    * A `stock-keeping unit (SKU)` is a scannable bar code, most often seen printed on product labels in a retail store. The label allows vendors to automatically track the movement of inventory. The SKU is composed of an alphanumeric combination of eight-or-so characters. The characters are a code that track the price, product details, and the manufacturer. SKUs may also be applied to intangible but billable products, such as units of repair time in an auto body shop or warranties.
    *  When a customer buys an item at the `point-of-sale (POS)`, the `SKU` is scanned and the `POS system automatically removes the item from the inventory` as well as recording other data such as the sale price. SKUs should not be confused with model numbers, although businesses may embed model numbers within SKUs.

* The design of `IMS` used a fairly simple data model called the **`hierarchical model`**, which has some remarkable similarities to the **`JSON model`** used by document databases [2]. It represented all data as a `tree of records nested within records`, much like the `JSON structure` of Figure 2-2.

* Like `document databases`, `IMS` worked well for `one-to-many relationships`, but it **`made many-to-many relationships difficult`**, and it didn’t **support joins**. Developers had to decide whether to duplicate (denormalize) data or to manually resolve references from one record to another. These problems of the `1960s` and `70s` were very much like the problems that developers are running into with `document databases today` [15].

* Various solutions were proposed to solve the limitations of the hierarchical model. The two most prominent were the `relational model (which became SQL, and took over the world)` and the `network model (which initially had a large following but eventually faded into obscurity)`. The “great debate” between these two camps lasted for much of the 1970s [2]. 

### The network model

* The `network model` was standardized by a committee called the `Conference on Data Systems Languages (CODASYL)` and implemented by several different database vendors; it is also known as the `CODASYL` model [16].

* The `CODASYL` model was a **generalization of the hierarchical model**. In the `tree structure of the hierarchical model`, every record has exactly **one parent**; in the `network model`, a record could have **multiple parents**. For example, there could be one record for the "Greater Seattle Area" region, and every user who lived in that region could be linked to it. **This allowed many-to-one and many-to-many relationships to be modeled.**

* The `links` between records in the `network model` were **not foreign keys**, but more `like pointers in a programming language` (while still being stored on disk). The only way of accessing a record was to `follow a path from a root record along these chains of links`. This was called an **`access path`**.
    * In the simplest case, an `access path` could be like the `traversal of a linked list`: start at the `head of the list`, and look at one record at a time until you find the one you want. But in a world of `many-to-many relationships`, several `different paths can lead to the same record`, and a programmer working with the network model had to keep track of these different `access paths` in their head.

* A query in `CODASYL` was performed by moving a `cursor through the database` by iterating over lists of records and following `access paths`. If a record had `multiple parents` (i.e., multiple incoming pointers from other records), the application code had to keep track of all the `various relationships`. Even `CODASYL` committee members admitted that this was like navigating around an `n-dimensional` data space [17].

* Although `manual access path` selection was able to make the most efficient use of the very limited hardware capabilities in the `1970s` (such as tape drives, whose seeks are extremely slow), the problem was that they made the code for `querying and updating` the database `complicated and inflexible`. With both the `hierarchical and the network model`, if you **didn’t have a path to the data you wanted, you were in a difficult situation**. You could change the `access paths`, but then you had to go through a lot of `handwritten database query code` and rewrite it to handle the new `access paths`. It was difficult to make changes to an application’s data model.

### The relational model

* What the `relational model` did, by contrast, was to lay out all the data in the open: a `relation` (table) is simply a `collection of tuples` (rows), and that’s it. There are no labyrinthine nested structures, no complicated access paths to follow if you want to look at the data. You can read any or all of the rows in a table, selecting those that match an arbitrary condition. You can read a particular row by designating some columns as a key and matching on those. `You can insert a new row into any table without worrying about foreign key relationships to and from other tables`.iv

* In a relational database, the query optimizer automatically decides which parts of the query to execute in which order, and which indexes to use. Those choices are effectively the `“access path,”` but the big difference is that they are **made automatically by the query optimizer**, not by the application developer, so we rarely need to think about them.
    * If you want to query your data in new ways, you can just `declare a new index`, and queries will **automatically** use `whichever indexes are most appropriate`. You don’t need to change your queries to take advantage of a new index. (See also “Query Languages for Data” on page 42.) The relational model thus made it much easier to add new features to applications.
    * But a key insight of the relational model was this: **you only need to build a query optimizer once, and then all applications that use the database can benefit from it**. If you don’t have a query optimizer, it’s easier to handcode the `access paths` for a particular query than to write a general-purpose optimizer—but the general-purpose solution wins in the long run.

### Comparison to document databases

* Document databases reverted back to the hierarchical model in one aspect: storing nested records (one-to-many relationships, like positions, education, and contact_info in Figure 2-1) within their parent record rather than in a separate table.

* However, when it comes to representing `many-to-one` and `many-to-many` relationships, relational and document databases are not fundamentally different: in both cases, the related item is referenced by a unique identifier, which is called a foreign key in the relational model and a document reference in the document model [9]. **That identifier is resolved at read time by using a join or follow-up queries.** To date, document databases have not followed the path of `CODASYL`.

### Relational Versus Document Databases Today

* There are many differences to consider when comparing relational databases to document databases, including their `fault-tolerance` properties (see Chapter 5) and `handling of concurrency` (see Chapter 7). In this chapter, we will concentrate only on the differences in the data model.

* The main arguments in favor of the document data model are `schema flexibility`, **better performance due to locality**, and that for some applications it is closer to the data structures used by the application. The `relational model` counters by providing `better support for joins, and many-to-one and many-to-many relationships`.

* The relational technique of **shredding**— splitting a document-like structure into multiple tables (like positions, education, and contact_info in Figure 2-1)—can lead to cumbersome schemas and unnecessarily complicated application code.

* However, if your application does use `many-to-many` relationships, the document model becomes less appealing. It’s possible to reduce the need for joins by `denormalizing`, but then the application code needs to do additional work to keep the `denormalized` data consistent.

* It’s not possible to say in general which data model leads to simpler application code; it depends on the kinds of relationships that exist between data items. For `highly interconnected data`, `the document model is awkward, the relational model is acceptable, and graph models (see “Graph-Like Data Models” on page 49) are the most natural`.

### Schema flexibility in the document model

* Most document databases, and the JSON support in relational databases, **do not enforce any schema on the data in documents**. XML support in relational databases usually comes with optional schema validation. No schema means that arbitrary keys and values can be added to a document, and when reading, clients have no guarantees as to what fields the documents may contain.

* Document databases are sometimes called `schemaless`, but that’s misleading, as the code that reads the data usually assumes some kind of structure—i.e., there is an `implicit schema`, but it is not enforced by the database [20]. A more accurate term is `schema-on-read` (the structure of the data is implicit, and only interpreted when the data is read), in contrast with `schema-on-write` (the traditional approach of relational databases, where the schema is explicit and the database ensures all written data con‐ forms to it) [21].

* Schema changes have a bad reputation of being slow and requiring downtime. This reputation is not entirely deserved: `most relational database systems execute the ALTER TABLE statement in a few milliseconds.` MySQL is a notable `exception`—it **copies the entire table on ALTER TABLE, which can mean minutes or even hours of downtime when altering a large table—although various tools exist to work around this limitation [24, 25, 26].**

* ![migration](./images/migration.png)

* The `schema-on-read` approach is advantageous if the items in the collection don’t all have the same structure for some reason (i.e., the data is heterogeneous)—for example, because:
    * There are many `different types of objects`, and it is `not practical to put each type of object in its own table`.
    * The `structure of the data is determined by external systems` over which you have no control and which may change at any time.

### Data locality for queries

* A document is usually stored as a `single continuous string`, encoded as `JSON, XML, or a binary variant thereof (such as MongoDB’s BSON)`. If your application often needs to access the entire document (for example, to render it on a web page), there is a `performance advantage to this storage locality`. If data is `split across multiple tables`, like in Figure 2-1, `multiple index lookups are required to retrieve it all`, which may require more disk seeks and take more time.
    * The locality advantage only applies if you need large parts of the document at the same time. The database typically needs to load the entire document, even if you access only a small portion of it, which can be wasteful on large documents. On updates to a document, the entire document usually needs to be rewritten—only modifications that don’t change the encoded size of a document can easily be performed in place [19]

* It’s worth pointing out that the idea of grouping related data together for locality is not limited to the document model. For example, `Google’s Spanner` database offers the same locality properties in a relational data model, by allowing the schema to declare that a table’s rows should be interleaved (nested) within a parent table [27]. Oracle allows the same, using a feature called multi-table index cluster tables [28]. The column-family concept in the Bigtable data model (used in Cassandra and HBase) has a similar purpose of managing locality [29].
    * Codd’s original description of the `relational model` [1] actually allowed something quite `similar to JSON documents` within a relational schema. He called it `nonsimple domains`. The idea was that a value in a row doesn’t have to just be a primitive datatype like a number or a string, but could also be a `nested relation (table)`—so you can have an arbitrarily nested tree structure as a value, much like the JSON or XML support that was `added to SQL over 30 years later`.

### Convergence of document and relational databases

* `Most relational database systems (other than MySQL) have supported XML since the mid-2000s`. This includes functions to make local modifications to XML documents and the ability to `index and query inside XML documents`, which allows applications to use data models very similar to what they would do when using a document database.
    * PostgreSQL since version 9.3 [8], MySQL since version 5.7, and IBM DB2 since ver‐ sion 10.5 [30] also have a similar level of support for `JSON documents`. Given the popularity of JSON for web APIs, it is likely that other relational databases will follow in their footsteps and add JSON support.
    * On the document database side, `RethinkDB` supports relational-like joins in its query language, and some `MongoDB` drivers automatically resolve database references (effectively performing a `client-side join`, although this is likely to be slower than a join performed in the database since it requires additional network round-trips and is less optimized).

### Query Languages for Data

* When the `relational model` was introduced, it included a new way of querying data: `SQL is a declarative query language`, whereas `IMS` and `CODASYL` queried the database using `imperative code`. What does that mean?

* Many commonly used programming languages are imperative. For example, if you have a list of animal species, you might write something like this to return only the sharks in the list:

* ![imperative](./images/imperative.png) 

* When SQL was defined, it followed the structure of the relational algebra fairly closely:

* ![declarative](./images/sql.png)

* An `imperative language` tells the computer to perform certain operations in a certain order. You can imagine stepping through the code `line by line`, evaluating conditions, updating variables, and deciding whether to go around the loop one more time.

* In a `declarative query language`, like `SQL` or `relational algebra`, you just specify the `pattern of the data you want`—what conditions the results must meet, and how you want the data to be transformed (e.g., sorted, grouped, and aggregated)—**but not how to achieve that goal**. It is up to the database system’s query optimizer to decide which indexes and which join methods to use, and in which order to execute various parts of the query.
    * A declarative query language is attractive because it is typically more concise and easier to work with than an imperative API. But more importantly, it also hides imple‐ mentation details of the database engine, which makes it possible for the database system to introduce performance improvements without requiring any changes to queries.

* **For example, in the imperative code shown at the beginning of this section, the list of animals appears in a particular order. If the database wants to reclaim unused disk space behind the scenes, it might need to move records around, changing the order in which the animals appear. Can the database do that safely, without breaking queries?**

* **The SQL example doesn’t guarantee any particular ordering, and so it doesn’t mind if the order changes. But if the query is written as imperative code, the database can never be sure whether the code is relying on the ordering or not. The fact that SQL is more limited in functionality gives the database much more room for automatic optimizations.**

* Finally, `declarative languages` often lend themselves to `parallel execution`. Today, `CPUs are getting faster by adding **more cores**, not by running at significantly higher clock speeds than before` [31]. Imperative code is very hard to parallelize across multiple cores and multiple machines, because it specifies instructions that must be performed in a particular order. **Declarative languages have a better chance of getting faster in parallel execution because they specify only the pattern of the results, not the algorithm that is used to determine the results.** The database is free to use a parallel implementation of the query language, if appropriate [32].

### Declarative Queries on the Web

* The `advantages of declarative query languages` are not limited to just databases. To illustrate the point, let’s compare declarative and imperative approaches in a completely different environment: **a web browser.**

* ![dom](./images/dom.png)

* Now say you want the title of the currently selected page to have a blue background, so that it is visually highlighted. `This is easy, using CSS`:

* ![css](./images/css.png) 

* Here the CSS selector `li.selected > p` declares the pattern of elements to which we want to apply the blue style: namely, all `<p>` elements whose direct parent is an `<li>` element with a CSS class of `selected`. The element `<p>Sharks</p>` in the example matches this pattern, but `<p>Whales</p>` does not match because its `<li>` parent lacks `class="selected"`.

* Here, the `XPath` expression `li[@class='selected']/p` is equivalent to the CSS selector `li.selected > p` in the previous example. What `CSS and XSL` have in common is that they are both `declarative languages` for specifying the styling of a document.
Imagine what life would be like if you had to use an imperative approach. In JavaScript, using the core Document Object Model (DOM) API, the result might look something like this:

* ![imperative_dom](./images/imperative_dom.png)

* This JavaScript imperatively sets the element `<p>Sharks</p>` to have a blue background, but the code is awful. Not only is it much longer and harder to understand than the CSS and XSL equivalents, but it also has some serious problems:

    * If the `selected` class is removed (e.g., because the user clicks a different page), the blue color won’t be removed, even if the code is rerun—and so the item will remain highlighted until the entire page is reloaded. With `CSS`, the browser `automatically` detects when the `li.selected > p` rule no longer applies and removes the blue background as soon as the selected class is removed.
    * If you want to take advantage of a new API, such as `document.getElementsBy ClassName("selected")` or `even document.evaluate()`—which may improve performance—you have to rewrite the code. On the other hand, browser vendors can improve the performance of `CSS and XPath without breaking compatibility`.

* **In a web browser, using declarative CSS styling is much better than manipulating styles imperatively in JavaScript. Similarly, in databases, declarative query languages like SQL turned out to be much better than imperative query APIs.vi**

### MapReduce Querying

* `MapReduce` is a programming model for processing `large amounts of data` in bulk across `many machines`, popularized by Google [33]. A limited form of `MapReduce` is supported by some `NoSQL` datastores, including `MongoDB and CouchDB`, as a mechanism for performing read-only queries across many documents.

* `MapReduce` is neither a `declarative query` language nor a fully `imperative query API`, but somewhere in between: the logic of the query is expressed with `snippets of code`, which are called repeatedly by the processing framework. It is based on the `map` (also known as `collect`) and `reduce` (also known as `fold` or `inject`) functions that exist in many functional programming languages.

* In PostgreSQL you might express that query like this:

* ![postgres_sql](./images/postgres_sql.png)

* The same can be expressed with `MongoDB’s MapReduce` feature as follows:

* ![mapreduce_mongo](./images/mapreduce_mongo.png)

* ![mapreduce_mongo2](./images/mapreduce_mongo2.png)

* The `map` function would be called once for each document, resulting in `emit("1995-12", 3) and emit("1995-12", 4)`. Subsequently, the `reduce function` would be called with `reduce("1995-12", [3, 4]), returning 7`.

* The `map` and `reduce` functions are somewhat restricted in what they are allowed to do. They must be `pure functions`, which means they `only use the data that is passed to them as input, they cannot perform additional database queries, and they must not have any side effects.` These restrictions allow the database to run the functions any where, in any order, and rerun them on failure. However, they are nevertheless powerful: they can parse strings, call library functions, perform calculations, and more.

* MapReduce is a fairly low-level programming model for distributed execution on a cluster of machines. Higher-level query languages like SQL can be implemented as a pipeline of MapReduce operations (see Chapter 10), but there are also many distributed implementations of SQL that don’t use MapReduce. **Note there is nothing in SQL that constrains it to running on a single machine, and MapReduce doesn’t have a monopoly on distributed query execution.**

* Moreover, a `declarative query language` offers more opportunities for a `query optimizer` to improve the performance of a query. For these reasons, MongoDB 2.2 added support for a `declarative query language called the aggregation pipeline` [9]. In this language, the same shark-counting query looks like this:

![aggregation_pipeline](./images/aggregation_pipeline.png)

* The `aggregation pipeline` language is similar in expressiveness to a `subset of SQL`, but it uses a `JSON-based syntax` rather than SQL’s English-sentence-style syntax; the difference is perhaps a matter of taste. The moral of the story is that a NoSQL system may find itself accidentally reinventing SQL, albeit in disguise.

### Graph-Like Data Models

* We saw earlier that `many-to-many` relationships are an important distinguishing feature between different data models. If your application has mostly `one-to-many` relationships (`tree-structured data`) or `no relationships` between records, the `document model` is appropriate.

* But what if many-to-many relationships are very common in your data? The relational model can handle simple cases of many-to-many relationships, but as the connections within your data become more complex, **it becomes more natural to start modeling your data as a graph.**

* **A graph consists of two kinds of objects: vertices (also known as nodes or entities) and edges (also known as relationships or arcs).**

* Facebook maintains a `single graph` with many different types of `vertices` and `edges`: vertices represent people, locations, events, checkins, and comments made by users; edges indicate which people are friends with each other, which checkin happened in which location, who commented on which post, who attended which event, and so on [35].