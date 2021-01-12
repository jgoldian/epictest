## Assumptions
A few assumptions are made in this architecture, which are worth calling out.

Daily and weekly leaderboards are based on a set period and have a global reset at the end of the day or week. This is different than a leaderboard that contains data from the last 24 hours or 7 days and is constantly rolling over. This is for a few reasons, the first being player expectation. Most (though not all) existing games have a roll over period. This helps players to understand the system without a lot of specific explanations or tutorials to them. The second being the design goal of a leaderboard system. Daily and weekly leaderboards help to create a player retention loop, in which there is something new for them to do every day or week. This builds habits around the game. While you do get this with a rolling window as well, it is harder to create specific events leaderboards or similar constructs. With a known window this can be taken into consideration for events or new game modes.

Total eliminations is a number that is constantly increasing after each game (even if by 0). This is different than a leaderboard that is tracking the best score from a game in that each game end will result in an update for the users scores under normal circumstances. This is contrasted to a leaderboard in a racing game in which only a user's BEST score is stored and it is important to compare them and update, instead of incrementing.

The leaderboard is not displayed to the user (with their position) immediately after game end. This allows for client side unloading, but also time for the system to process without the user being effected. While this can be worked around if so desired, this assumption allows for local write caching and an increase in scalability.

There is a userId that hashes to a relatively even distribution as it will be used as a primary key.

There is a configuration service that allows for updates during runtime that can be pushed to the services that subscribe to the updates.

Analytics is stored in another system and not considered or discussed for this solution, other than that the services would write to it.

The backend game service that is pushing the score is secure and accurate. No anti-cheat systems are presented in this design.

There is a way to get the users full score list, but it is not during the main game loops.

New leaderboards (monthly for example) or game modes can be added. However, the assumption is that the time to create the in game UI will take longer to create than a server update. 

The leaderboard is of size 100, though the architecture can be changed to support unlimited size if required. By having it be top 100, the lists can be reduced after insert, which increases performance and allows them to run on a smaller server. However by scaling up the server instance this limitation can be increased or even removed depending on the design of the system. This could be changed per game mode and per leaderboard type.

Assumption that the cloud provider is AWS, though no specific architecture decisions require AWS and there are equivalent techs on all major cloud providers.

## Glossary

Reset - The time at which the leaderboards need to reset. Refers to both the weekly reset and the daily reset.

Reset window - The period in between resets.

Instance - A single instance of a compute unit executing a service. This could be an EC2 instance, a physical machine or a VM.

## High Level Architecture

![High level architecture diagram](/Architecture.png)

Note: This approach is overengineered for the specific problem that is being asked. To accomplish what is asked the UserScoreDataStore is not needed and there is no need for 2 microservices. However, based on experience there is likely a desire to be able to retrieve a user's information, expand on their metadata about the matches, etc. This design gives flexibility to those goals without sacrificing the performance of the leaderboard service itself.

### ConfigurationService
 A service that allows for configuration to be pushed to the entire fleet of services and instances. There are many examples of such services, for example AWS Config as a public cloud offering that could operate in this manner. 

### JobScheduler
A service that allows for jobs to be scheduled and executed on. This could be a custom service for this (if one exists) or a serverless function that is scheduled to run. For this architecture it will be assumed to be a proxy for AWS Lambda functions.

Three jobs will be scheduled for each reset.

The first will run before the reset, approximately 10 minutes before hand (though that can be changed without significant impact). It will create the new table to be written to for each game mode. The name will follow the format of `{GameMode}-{CreationDay}-{Type}-UserScoreData` and `{GameMode}-{CreatationDay}-{Type}-Leaderboard`. This will allow for new tables to be created without name collisions before they are needed.

The second will be the reset job itself, which will update the configuration service with the new name of the table.

The third will be the cleanup job. After the table is no longer in use it can be archived. This would also be where any rewards could be calculated.

Another job that would be scheduled would be to reduce the size of the LeaderboardStore periodically. Given that each insert is going to be log N where N is the number of members, purging unnecessary members will keep the system running faster and require smaller instances to run the service. For fully ranked leaderboards this would not be used.

### UserScoreDataService
A service that stores and retrieves user data. It uses UserScoreDataStore for its storage and writes to the leaderboard service.

Scalability - As this service is stateless, it can be autoscaled based on the current load.
Reliability - The service should have standard monitoring in place to ensure it is functioning properly. This includes looking for large deviations in the number of requests per second coming in, the number of responses as well as overall machine metrics, such as CPU and memory usage. Because each instance is disposable, if there are any issues with a given instance the solution to most issues with an instance is to create a new one and retire the old one. 
Resilience - If the user data store is no longer available for any reason, the first thing to do would be to retry. If that doesn't work there are a few options that can be considered. The first is a secondary store to maintain the list of unprocessed updates. This could be in local memory or in a distributed place. The second option would be to drop the data if the request cannot be processed and log the information in a specific error message. This would allow for a post-processing job to potentially pick it up and process it offline for the overall leaderboard. The second option would be preferable if errors should be rare, but in high error rate scenarios the first would be a better choice.
Capacity - This service should be mostly bound by the UserScoreDataStore, but locally by its CPU. In a cloud environment it should be on a relatively small instance that can be distributed world wide and allow for more redundancy. However, finding the sweet spot for which instances make the most sense based on price would require further testing.

See the [API specification ](https://app.swaggerhub.com/apis/JeffGold/LeaderboardService/1.0.1#/UserScoreData) for exact APIs.

### UserScoreDataStore
A set of databases that store the users current scores for each game mode. There are separate tables for each game mode and each reset window. This means that in order to retrieve the full list of scores for a user, data would need to be pulled from 12 different tables (3 reset windows * 4 game modes). On write to the user data, the current value is returned to allow that data to flow downstream.

Lazy updates vs daily resets
Two approaches were considered for this database for handling the resets that occur. One approach is to record the last time a user has updated their score, read that in and then determine if the daily and weekly need to be reset. This allows for all of a users score data to be stored in one row for any given game mode; this gives the advantage of a single read and write per game session (per user). However it requires each app tier service to know if the reset should happen. While it is possible to use a time service or other solution for this, the bugs associated with it are difficult to reproduce or track down as they are on 1 server out of many in the fleet and not related to the code itself, but the machine configuration. It also does not allow for daily / weekly rewards to easily be processed after as the scores are updated as needed, reducing the systems overall flexibility.

Instead the approach that was decided on is to create a new table on every reset. The table will be pre-created before the reset by the scheduled job service. On reset, the JobScheduler will then update the configurationService with the name of the new UserScoreDataStore for that time period. The UserScoreDataService than always writes to the current UserScoreDataStore as a increment to its current score, creating the row if needed.

The user data store has a few characteristics that are important when deciding the storage solution for it. First is that it must scale based on the numbers of concurrent users, as after every match updates will need to be made for each and every player. Thus a data store that will allow for a partitioning based on a userID is required. Second is that a data store that allows for increment operators as a primitive is important. As most operations will be to add the users eliminations for that match to their current score, not having to retrieve the value from the store, add it in memory on the app tier and then set it will allow for a much more scalable system. Further, if the store allows for conditional operations, such as creating the row and setting the value if it is not created, that will simplify the app tier logic, without adding much in terms of scalability requirements to the database.

The biggest decision point for which technology to look at for this database is the transactional nature of the system and its requirements. As each game will require writes to 3 different tables, the biggest question is if this should be in a transactional database that will allow a single transaction across the tables. The advantage of taking a transaction is that it would mean the data is consistent with itself in the case of a write failure. However this would come at the cost of speed, as taking a transaction across tables is an expensive action. It also means that if any of the tables is down, all of the writes would fail. To that end the decision was made that it the data inconsistency was acceptable and not worth the price of a transactional system.

Based on these considerations a nosql key-value pair database is the choice. For AWS this would mean DynamoDB being the best choice as it allows for simple queries, has the ability to use the userID as its primary key, and allows for dynamic scaling as the game grows.

Each write would be an update with that matches score, with a conditional that creates the row with a 0 value if it doesn't exist. The row would contain the userId, the currentScore and any metadata associated with the score. On return it would return the row so that the value can be sent to the LeaderboardService.

Scalability - As the store is a separate table per gametype-reset window and all actions are against the primary key the store itself can be distributed across any number of servers.
Capacity planning - Given that the number of writes would be known based on the number of games per second that are finishing * the number of users per game, a scale unit can be applied to the DynamoDB instance. This creates a clear cost amount per day for the feature.

### LeaderboardService

 A service that manages reading and writing the leaderboards themselves.

Scalability - As this service has no state, it can be autoscaled based on the current load.
Reliability - The service should have standard monitoring in place to ensure it is functioning properly. This includes looking for large deviations in the number of requests per second coming in, the number of responses as well as overall machine metrics, such as CPU and memory usage. As the service is non-stateful, it is best to create a new instance if there are concerns.
Resilience - If the leaderboard store is unavailable errors should be returned to the caller, as well as logged. As this data store is not a source of truth for information, once the store is back up and either a game is finished (in the case of a write) or the request is retried, the correct response will be given.
Capacity - This service should be mostly bound by the LeaderboardStore. In terms of local resource it is likely either limited by the CPU or the NIC. In a cloud environment it should be on a relatively small instance that can be distributed world wide and allow for more redundancy. However, finding the sweet spot for which instances make the most sense based on price would require further testing.

 See [API reference](https://app.swaggerhub.com/apis/JeffGold/LeaderboardService/1.0.0#/User/getLeaderboard) for more details.

### LeaderboardStore -
A set of tables, 1 per reset window per game mode.

In this architecture the LeaderboardStore has some interesting characteristics that help determine which storage solution makes the most sense for it. First is that it does not scale based on users directly for writes, as there is a buffer created by the LeaderboardService. Instead there is a constant amount of writes, which is based on the number of instances of the LeaderboardService. Reads will be based on the number of displays for the leaderboard. Because of the buffering in the Service layer each write will be a batch, with 100 entires to be compared to the existing board and merged in, keeping only the top 100 scores. While this could be done at the services layer, as the leaderboard could be updated by the time the read, merge and write finishes on the service layer this creates a consistency issue. Thus the database itself must be able to merge the scores and maintain that consistency.

Redis has a Sorted Set functionality that allows for multiple inserts, the increment or add functionality and deals with the rank as part of the insert. Coupled that with the ability to remove a range by rank to trim down the server if required it meets all the requirements for this database. AWS Elasticache creates a managed service which is protocol compatible with Redis. However it is possible to run the service on a VM as well, if so desired.

Scalability -	While Redis is a very scalable system, the sorted set function means that it can only be scaled up and not out. If the number of reads and writes is greater than what the largest server that can be used, either for cost or availability reasons, than the system will fall apart.  To solve that scores would not be written directly to the database from the LeaderboardService, but instead through an in memory buffer. This buffer would accumulate data for a configurable amount of time, approximately 1-2 seconds and then write the data to the LeaderboardStore. By creating a double buffered system, this can be accomplished without locking. Every second the system would do the following actions:
 1) Switch the current buffer with the secondary buffer.
 2) Reduce the current buffer to the top 100 as there is no reason to send data that the service knows cannot possibly be on the leaderboard.  This would be removed in a full ranked leaderboard.
 3) Send the data to the current LeaderboardStore tables. This would be done using pipelining to reduce server load.
 4) Reduce the size of the LeaderboardStore tables to the top 100, to reduce the growth of the table. This would be removed for a full ranked of the leaderboard.
 5) Reset the buffer to be clean

Capacity planning - Given that the number of writes would be known based on the number of games per second that are finishing * the number of users per game, this will help dictate the size of the Redis instance as well as if the scalability factors above are needed. As a SortedSet must be on a single machine though, this solution requires a scale up approach, resulting in more costs.


## API Specification

[Link to swagger](https://app.swaggerhub.com/apis/JeffGold/LeaderboardService/1.0.0)
