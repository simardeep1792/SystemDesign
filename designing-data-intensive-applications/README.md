# Designing Data Intensive Applications

## Reliable, Scalable, and Maintainable Applications

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