# Design a Rate Limiter

## Definition
Control the rate of traffic sent by client or service. Excessive calls over threshold is blocked.

## Goal
1. Prevent resource exhaustion (e.g. DoS attack)
1. Reduce cost - fewer servers, less paid 3rd-party API usage
1. Prevent server overload

## Clarifications
1. Type of Rate Limiter: client-side vs. server-side vs. middleware gateway - similar question: is rate limiter a separate service, or inside the server code
1. Throttle rule: based on user id vs. based on IP etc.
	* this determine how many buckets/counters will we have - one for each API endpoint, plus one global bucket shared by all requests
1. Scale of the system: start-up vs. large company
1. Distributed system or not
1. Do we inform users when throttled

## Trade-off #1 - Rate Limiter Type

### Server-side
* if we are we have a tech stack that is efficient for server-side implementation - use server-side
* if we are picky about the algorithm, use server-side because middleware has limited choices for algorithm

### Middleware
* if we already used microservice architecture, or even already using an API gateway
* if we want to save time - commercial API gateways comes handy

### Client-side
* client-side is usually not recommended because:
	* unreliable - requests can easily be forged
	* we have no control over the implementations

## Trade-off #2 - Algorithm Type

To memorize, there are two "buckets", two "window counters", and one "window log"

### Token Bucket
* Constant rate of refilling capacity
* Params
	* Bucket size
	* Refill rate
* Pros
	* Easy to implement
	* Memory efficient (only two params)
	* Allow a burst of traffic in a short time (flash sale)
* Cons
	* Hard to tune the only 2 params properly

### Leaking Bucket
* Constant rate of processing the queue
* Params
	* Bucket (queue) size
	* Outflow rate
* Pros
	* Memory efficient (a limited queue size)
	* Allow a stable outflow if needed
* Cons
	* Not good for a burst of requests
	* Hard to tune the only 2 params properly

### Fixed Window Counter
* Resetting quota at the end of each unit time window (every full minute)
* Params
	* Defined unit time window (e.g. daily from midnight to midnight)
	* Quota size
* Pros
	* Fits certain use cases
* Cons
	* A spike in traffic near the edges of a window could cause more requests than the allowed quota to go through

### Sliding Window Counter
* Resetting quota at the end of each rolling window (any time). At each timestamp, define \#Requests = \#Requests in Previous Window * Overlap\% + \#Requests in Current Window 
* Params
	* The rolling window length (e.g. one minute)
	* Quota size (number of allowed requests)
* Pros
	* Smooths out spikes of requests
	* Memory efficient
* Cons
	* Only an approximation because we assume requests in the previous window are evenly distributed

## Deep Dive Q&A

### Where is RL counters stored?
* Recommend to store at in-memory cache, such as Redis. Reasons:
	* Fast access
	* Support time-based expiration strategy
	* Redis API supports: `INCR`, which increments the counter by 1, and `EXPIRE`, which expires the data and deletes it
* Not recommend to store counters on database, because disk access is slow


### How to create a RL rule?
* Example
	* Domain (e.g. auth, messaging)
	* Key-value (e.g. type: login or marketing)
	* Rate limit
		* Unit (e.g. day or minute)
		* Requests per unit (e.g. 5)


### Where is RL rules store?
* Unlike counters, rules need to be more persistent, so it is store on disk.
* We can still cache those frequently accessed rules.


### How to notify clients about throttling?
* Use HTTP response header to include RL information
* Information include:
	* \#Remaining - how many allowed requests left
	* \#Limit - how many total requests allowed per unit time
	* \#Retry-After - how many seconds to wait until the next retry without being throttled


### Challenges of having a distributed environment
* Race condition (when write)
	* Two requests concurrently read the same counter before either of them writes the value back. Assuming originally counter value is 3, both of them would increment it, and thought it would be incremented to 4, but in fact, it should be 5.
	* Solutions
		* Sorted sets data structure in Redis
		* Lua script
		* Lock is infeasible, because it significantly slows down the system
* Synchronization issue (when read)
	* Clients access different RL server randomly. One server might not have the updated data for another client (stale)
	* Solutions
		* Use a centralized data store, e.g. Redis
		* Sticky session is infeasible - it would allow a client to always send traffic to the same RL, but that is not scalable


### How to further improve performance?
* Use multi-data center setup, allow traffic to be automatically routed to a nearest edge server to reduce latency
* Synchronize using an eventual consistency model


### How to monitor?
* The goal is to check if:
	* RL algorithm is effective
	* RL rules are effective
* We can track metrics such as the number of limited requests

## Final Design
### Diagram
TBD

### Steps
1. Client send requests to rate limiter
1. RL fetches the counter from Redis, while fetching rules from Cache
1. If the request has exceeded the limit, reject
	* send 429 HTTP code back to client
	* move the request to a queue to be processed later
1. If allowed by counter
	* send request to servers (e.g. or load balancer)
	* increment the counter and save it back to Redis
	