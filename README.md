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

## 8. Bottlenecks

## 9. Security and Permissions

