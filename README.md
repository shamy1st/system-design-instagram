# Instagram Design

Design a photo-sharing service like Instagram, where users can upload photos to share them with other users.

Similar Services: Flickr, Picasa

Difficulty Level: Medium

## What is Instagram?

 * Instagram is a social networking service which enables its users to upload and share their photos and videos with other users.
 * Instagram users can choose to share information either publicly or privately.
 * Anything shared publicly can be seen by any other user, whereas privately shared content can only be accessed by a specified set of people.
 * Instagram also enables its users to share through many other social networking platforms, such as Facebook, Twitter, Flickr, and Tumblr.
 * For the sake of this exercise, we plan to design a simpler version of Instagram, where a user can share photos and can also follow other users.
 * The ‘News Feed’ for each user will consist of top photos of all the people the user follows.

## 1. Requirements

### Functional Requirements

1. Users should be able to upload/download/view photos.
2. Users can perform searches based on photo/video titles.
3. Users can follow other users.
4. The system should be able to generate and display a user’s News Feed consisting of top photos from all the people the user follows.

### Non-functional Requirements

1. Our service needs to be highly available.
2. The acceptable latency of the system is 200ms for News Feed generation.
3. Consistency can take a hit (in the interest of availability), if a user doesn’t see a photo for a while; it should be fine.
4. The system should be highly reliable; any uploaded photo or video should never be lost.

### Out of Scope Requirements

* Adding tags to photos, searching photos on tags, commenting on photos, tagging users to photos, who to follow, etc.

## 2. Design Considerations

The system would be read-heavy, so we will focus on building a system that can retrieve photos quickly.

1. Practically, users can upload as many photos as they like. Efficient management of storage should be a crucial factor while designing this system.
2. Low latency is expected while viewing photos.
3. Data should be 100% reliable. If a user uploads a photo, the system will guarantee that it will never be lost.

## 3. Database Model

![](https://github.com/shamy1st/system-design-instagram/blob/main/instagram-database-model.png)

* We need to have an index on (PhotoID, CreationDate) since we need to fetch recent photos first.
* We can store photos in a distributed file storage like [HDFS](https://en.wikipedia.org/wiki/Apache_Hadoop) or [S3](https://en.wikipedia.org/wiki/Amazon_S3).
* We can store the above schema in a distributed key-value store to enjoy the benefits offered by NoSQL. All the metadata related to photos can go to a table where the ‘key’ would be the ‘PhotoID’ and the ‘value’ would be an object containing PhotoLocation, UserLocation, CreationTimestamp, etc.
* We need to store relationships between users and photos, to know who owns which photo.
* We also need to store the list of people a user follows.
* For both of these tables, we can use a wide-column datastore like [Cassandra](https://en.wikipedia.org/wiki/Apache_Cassandra).
* For the ‘UserPhoto’ table, the ‘key’ would be ‘UserID’ and the ‘value’ would be the list of ‘PhotoIDs’ the user owns, stored in different columns.
* We will have a similar scheme for the ‘UserFollow’ table.
* Cassandra or key-value stores in general, always maintain a certain number of replicas to offer reliability.
* Also, in such data stores, deletes don’t get applied instantly, data is retained for certain days (to support undeleting) before getting removed from the system permanently.

## 4. Estimation

### Traffic Estimates

### Storage Estimates

* Let’s assume we have 1000M total users, with 500M daily active users.
* 95M new photos per day, 1100 new photos per second.
* Average photo file size => 200KB
* Total space required for 1 day of photos: 95M * 200KB ~ 19TB
* Total space required for 10 years: 19TB * 365 * 10 ~ 70PB

* Let’s estimate how much data will be going into each table and how much total storage we will need for 10 years.
* **User**: Assuming each “int” and “dateTime” is four bytes, each row in the User’s table will be of 68 bytes:
  * UserID (4 bytes) + Name (20 bytes) + Email (32 bytes) + DateOfBirth (4 bytes) + CreationDate (4 bytes) + LastLogin (4 bytes) = 68 bytes
  * If we have 1000 million users, we will need 68GB of total storage: 1000 million * 68 ~= 68GB
* **Photo**: Each row in Photo’s table will be of 284 bytes:
  * PhotoID (4 bytes) + UserID (4 bytes) + PhotoPath (256 bytes) + PhotoLatitude (4 bytes) + PhotLongitude(4 bytes) + UserLatitude (4 bytes) + UserLongitude (4 bytes) + CreationDate (4 bytes) = 284 bytes
  * If 95M new photos get uploaded every day, we will need 27GB of storage for one day: 95M * 284 bytes ~= 27GB per day
  * For 10 years we will need 99TB of storage.
* **UserFollow**: Each row in the UserFollow table will consist of 8 bytes. If we have 1000 million users and on average each user follows 500 users. We would need 4TB of storage for the UserFollow table: 1000 million users * 500 followers * 8 bytes ~= 4TB
* **Total space** required for all tables for 10 years will be 103TB: 68GB + 99TB + 4TB ~= 103TB

### Bandwidth Estimates

### Memory Estimates

### High-level Estimates

Metric               | Estimate
---------------------|---------
New Photos           | 1100/s
Incoming data        | -- KB/s
Outgoing data        | -- MB/s
Storage for 10 years | 70 PB
Memory for cache     | -- GB

## 5. High-level Design

* At a high-level, we need to support two scenarios, one to upload photos and the other to view/search photos.
* Our service would need some object storage servers to store photos and also some database servers to store metadata information about the photos.

![](https://github.com/shamy1st/system-design-instagram/blob/main/instagram-hld.png)

## 6. System APIs



## 7. Low-level Design

![](https://github.com/shamy1st/system-design-instagram/blob/main/lld.png)

* Photo uploads (or writes) can be slow as they have to go to the disk, whereas reads will be faster, especially if they are being served from cache.
* Uploading users can consume all the available connections, as uploading is a slow process.
* This means that ‘reads’ cannot be served if the system gets busy with all the write requests.
* We should keep in mind that web servers have a connection limit before designing our system.
* If we assume that a web server can have a maximum of 500 connections at any time, then it can’t have more than 500 concurrent uploads or reads.
* To handle this bottleneck we can split reads and writes into separate services.
* We will have dedicated servers for reads and different servers for writes to ensure that uploads don’t hog the system.
* Separating photos’ read and write requests will also allow us to scale and optimize each of these operations independently.

## 8. Bottlenecks

### Reliability and Redundancy

* Losing files is not an option for our service.
* Therefore, we will store multiple copies of each file so that if one storage server dies we can retrieve the photo from the other copy present on a different storage server.
* This same principle also applies to other components of the system.
* If we want to have high availability of the system, we need to have multiple replicas of services running in the system, so that if a few services die down the system still remains available and running.
* Redundancy removes the single point of failure in the system.
* If only one instance of a service is required to run at any point, we can run a redundant secondary copy of the service that is not serving any traffic, but it can take control after the failover when primary has a problem.
* Creating redundancy in a system can remove single points of failure and provide a backup or spare functionality if needed in a crisis. 
* For example, if there are two instances of the same service running in production and one fails or degrades, the system can failover to the healthy copy.
* Failover can happen automatically or require manual intervention.

### Data Sharding

#### a. Partitioning based on UserID

* Let’s assume we shard based on the ‘UserID’ so that we can keep all photos of a user on the same shard.
* If one DB shard is 1TB, we will need 103 shards to store 103TB of data.
* Let’s assume for better performance and scalability we keep 110 shards.
* So we’ll find the shard number by UserID % 110 and then store the data there. 
* To uniquely identify any photo in our system, we can append shard number with each PhotoID.

* How can we generate PhotoIDs? 
  * Each DB shard can have its own auto-increment sequence for PhotoIDs and since we will append ShardID with each PhotoID, it will make it unique throughout our system.

* What are the different issues with this partitioning scheme?
  * How would we handle hot users? Several people follow such hot users and a lot of other people see any photo they upload.
  * Some users will have a lot of photos compared to others, thus making a non-uniform distribution of storage.
  * What if we cannot store all pictures of a user on one shard? If we distribute photos of a user onto multiple shards will it cause higher latencies?
  * Storing all photos of a user on one shard can cause issues like unavailability of all of the user’s data if that shard is down or higher latency if it is serving high load etc.

#### b. Partitioning based on PhotoID

* If we can generate unique PhotoIDs first and then find a shard number through “PhotoID % 10”, the above problems will have been solved. We would not need to append ShardID with PhotoID in this case as PhotoID will itself be unique throughout the system.

* How can we generate PhotoIDs? 
  * Here we cannot have an auto-incrementing sequence in each shard to define PhotoID because we need to know PhotoID first to find the shard where it will be stored. One solution could be that we dedicate a separate database instance to generate auto-incrementing IDs. If our PhotoID can fit into 64 bits, we can define a table containing only a 64 bit ID field. So whenever we would like to add a photo in our system, we can insert a new row in this table and take that ID to be our PhotoID of the new photo.

* Wouldn’t this key generating DB be a single point of failure? 
  * Yes, it would be. A workaround for that could be defining two such databases with one generating even numbered IDs and the other odd numbered. For the MySQL, the following script can define such sequences:
  * We can put a load balancer in front of both of these databases to round robin between them and to deal with downtime. Both these servers could be out of sync with one generating more keys than the other, but this will not cause any issue in our system. We can extend this design by defining separate ID tables for Users, Photo-Comments, or other objects present in our system.
  * Alternately, we can implement a ‘key’ generation scheme similar to what we have discussed in Designing a URL Shortening service like TinyURL.

* How can we plan for the future growth of our system? 
  * We can have a large number of logical partitions to accommodate future data growth, such that in the beginning, multiple logical partitions reside on a single physical database server. Since each database server can have multiple database instances on it, we can have separate databases for each logical partition on any server. So whenever we feel that a particular database server has a lot of data, we can migrate some logical partitions from it to another server. We can maintain a config file (or a separate database) that can map our logical partitions to database servers; this will enable us to move partitions around easily. Whenever we want to move a partition, we only have to update the config file to announce the change.

### Cache and Load balancing

* Our service would need a massive-scale photo delivery system to serve the globally distributed users. 
* Our service should push its content closer to the user using a large number of geographically distributed photo cache servers and use CDNs (for details see Caching).
* We can introduce a cache for metadata servers to cache hot database rows. 
* We can use Memcache to cache the data and Application servers before hitting database can quickly check if the cache has desired rows. 
* Least Recently Used (LRU) can be a reasonable cache eviction policy for our system. 
* Under this policy, we discard the least recently viewed row first.
* How can we build more intelligent cache? If we go with 80-20 rule, i.e., 20% of daily read volume for photos is generating 80% of traffic which means that certain photos are so popular that the majority of people read them. 
* This dictates that we can try caching 20% of daily read volume of photos and metadata.

## 9. Security and Permissions


