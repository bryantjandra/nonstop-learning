
**Functional Requirements**: What is the functionality of your application? (e.g. what screens and buttons are there and what each does?)

**Nonfunctional Requirements**: Seemingly obvious requirements but just as important as FRs (e.g. how fast, reliable, secure, legally compliant, and maintainable is our app?)

*Examples of NFRs*: 
1. measuring the performance of a system
2. system's reliability (continuing to work correctly, even when things go wrong), 
3. scalable (having efficient ways of adding computing capacity as the load on the system grows). 
4. maintainable system in the long term

## Case Study: Implementing X (formerly Twitter)

*This simplified X allows*:
- Users to post messages
- Users can follow other users

*Assumptions*:
- Total of 500 million posts/day OR 5,800 posts/second.
	- The above rate can spike to 150,000 posts/second
- Average user follows 200 people and has 200 followers.

There would be 3 tables:
1. Users (id, screen_name, profile_image)
2. Posts (id, sender_id, text, timestamp), sender_id is what user made the psot
3. Follows (follower_id, followee_id), follower_id is the currently logged in user whilst followee_id is the id of the user that this logged in user follows

The main read operation: Home Timeline (displays recent posts by people the user is following). 

Potential Query:

```SQL
SELECT posts.*, users.* FROM posts
	JOIN follows ON posts.sender_id = follows.followee_id
	JOIN users ON posts.sender_id = users.id
	WHERE follows.follower_id = current_user
	ORDER BY posts.timestamp DESC
	LIMIT 1000 
```


Posts are supposed to be *timely* --> Maybe we can repeat the above query every 5 seconds for each user. 

If we assume just 10 million users are online and logged in --> that would mean running the query **2 million times / second** (THIS IS ALOT!).

The query is also **quite expensive**. 
- If a user follows 200 people, the query needs to fetch a list of recent posts by each of those 200 people and merge those lists. 
- This means two million queries * 200 followed accounts = 400 million lookups/second.


### How could we do better then?

**Stop polling**:
1. Server should actively push new posts to any followers who are currently online. 
2. We should precompute the results of the query such that a user's request for their home timeline can be served from a cache.

For each user, we store a data structure (e.g. cache) containing their home timeline (posts of people they are following). This **cache** serves as a *materialized view*. 

Every time a user makes a post, we look up all their followers and insert that post into this data structure of every follower. Thus, whenever a user logs in, we can simply hand them this precomputed home timeline. This **process** is known as *materialization*.

Moreover, to receive a notification about any new posts on their timeline, the user's client simply needs to subscribe to the stream of posts being added to their data structure.


**Downside**: Need to do more work every time a user makes a post.

When one initial request results in several downstream requests being carried out --> **fan-out** (the factor by which the number of requests increases). 

*if the average post* reaches 200 followers (fan-out factor of 200), and ar a rate of 2,800 posts/second, we will still need to do >= 1,000,000 writes/second (still alot but better than previous 400,000,000 lookups/second). 




## Describing Performance

There are *2 main types* of metrics when discussing software performance.

1. **Response Time**: Elapsed time from the moment when a user makes a request until they received the response. 
2. **Throughput**: Number of requests per second, or data volume per second, that the system is processing. For a given allocation of hardware resources, there's a maximum throughput that can be handled. 

![[ThroughputResponseRelationship.png]]

As shown in the figure above, a *service* usually has a low response time when throughput is low. But, response times increases as load increases. 

This is primarily because of **queueing**. When a request arrives in a highly loaded system, the CPU Is likely already in the process of handling an earlier request, and therefore the incoming request needs to wait until the earlier request has been completed. 

> As throughput approaches the maximum that **the hardware can handle**, queueing delays increase sharply!


 