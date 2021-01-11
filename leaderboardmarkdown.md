
###### What's Leader Board Problem
   A leader board is the most commonly used game feature across multi player games. 
Though leader board can be of various types and some times customized as per requirement 
but basic tenant remains same i.e. to rank the players.
   It’s a collection of high scores achieved in a game session during a specific time segment in a specific portion of a 
game for a specific set of users. It is used to increase the level of competition amongst players by ranking them in 
a variety of ways with the aim of generating more game play.
   Let’s build a minimal player-facing leader board for one of Epic’s multi player online games.

Leader board should be based on , 

**Time segment** relates to the lifetime of the leader board. It is permanent or resets every day/week/month.
Lets build the one with daily/weekly/all-time aggregations.

**A specific set of users** relates to the ‘locality’ of the users and means that the leader board may contain 
scores for users that are in a specific region (e.g. Asia, Europe) or worldwide
Let's build the leader board for players from all the regions and ability to build it on region basis as well.

**Game portion** relates to the portion/segment/mode  of the game the score was achieved in. 
The score is relevant to the entire game or in just one of its levels? Is the score relevant to a single round 
of game play (single session) or multiple ones?

Lets build the leader board with 4 game modes.


###### Scoring 
 A score is usually a number, but the scoring order could be of 2 types. 
 e.g. the higher score could be better than the one with the lower score or vice-versa (Number of seconds taken by the player to finish the game.).
      Also, there could be different types of score operations. e.g. Adding the new score into the old score (Cumulative scoring) or replacing the 
      existing score with a new one.
 
 for this exercise we will consider a score as number which could be cumulatively added and highest score results in higher ranks.
 
######  Ranking the members 
 Default ranking provides a unique rank to each member. If members having the same score, they are ranked lexicographically.
 Tie ranking provides the same rank to the members with the same score.
 Competition ranking provides the same rank to the members with the same score, but a gap is left in the ranking numbers. 
 
 for this exercise we will go with Default Ranking.
 
 ####### Load on the system
 10 million concurrent players
 10,000 matches end every minute
  
 ###### Leader board data needs to be persistent.
 
 ###### Solution Architecture , API specs , Data Model , Tech Stack , Scalability , Reliability , resilience , Capacity Planning. 
 
 _Solution Architecture:_ 
 There would different ways of approaching solution for this , I have tried to design this solution around Redis enterprise which looks good match for the 
 performance expectations. Recently release Redis 6 seems to have been made available keeping these kind of use cases in mind. Please find architecture im LeaderBoardComponenetsDiagram.pdf
 Solution below tries to explain the drawn architecture.
 
 _API Specs:_
 Looking at the scalability and performance demands of the system , it would be elegant if high stream of match results 
 are posted using  Redis streams publishers than through Rest API, as REST API will have overhead. As requested in the problem document , 
 I am keeping match results post in REST as well.
 Please find **leaderBoardRest.html** for REST spec defined using swagger OAS3.
 
 _Data Model:_
 The proposed design will rely on Redis SortedSet data structure and snapshot disk store , 
 the key value data model is for inserting match events that too in async mode. If needed for querying this data 
 Redis Multi Modal capabilities like RedisSearch, RedisGraph, RedisJSON, RedisTimeSeries, and RedisAI etc could be brought in effect.
 
 
 Key Value Entities :-
     
     **Player Information**
      'player:0:playerId' ='100'
      'player:0:name' ='Andy Richard'
      'player:0:age' ='20'
      'player:0:locationId' = 'EU_112'
      
      **Match Information**
      'match:1:matchId'='123'
      'match:1:matchName'='AAAASeries2020'
      'match:1:matchMode'='4'
      'match:1:playedDate'='2021-01-01T14:47:00.000Z'
      'match:1:scores'='[playerScore:0,playerScore:1, playerScore:2]'
      
      **Score Information**   {Entities have some repetative information to scale the score processing in case if 
      system is directly processing scores to catch up time loss during recovery , Key value data bases way...}
      'playerScore:0:score'='10000'
      'playerScore:0:playedDate'='2021-01-01T14:47:00.000Z'
      'playerScore:0:playerId'='100'
      'playerScore:0:locationId'='1'
      'playerScore:0:groupId'='188'
      'playerScore:0:matchName'='AAAASeries2020'     
      'playerScore:0:matchMode'='4'     
                  
      **Leader Result**  {This result will be disk snapshot saved by Redis Enterprise's sorted set which lives 
      in swappable memory to disk as the demand changes using Redis on Flash to support large datasets ,  
      always ordered by score.
      Redis enterprise scales to millions of operations per second with <1ms sync across globe, 
      provides 99.999% database uptime based on built in durability and single digit second failover.
      With average hardware millions of records would be manipulated and saved by Redis Enterprize}
      
      **Rank Information**
        'aggregation:daily:Id'='1'
        'aggregation:daily:matchMode'='1234'
        'aggregation:daily:location'='locationId'
        'aggregation:daily:group'='groupId'
        'aggregation:daily:updatedDate'='2021-01-01T14:47:00.000Z'
        'aggregation:daily:ranks'='[playerRank:1, playerRank:2 .... playerRank:5000]'
    
      Soreted sets are efficient with ranks -  name value pairs (in RAM and disk memory swap) , No need to put upper cap of 100 players here ,
      Redis promises it could be billions on average hardware.
       
      players in cache which will swapped with disk.
      **Player Rank information**
      'rank:1:name'='Andy Cohel'
      'rank:1:score'='1000000000000'
               
               
  _Tech Stack_  
  
   1. Redis 6 is a in-memory data structure store with write-behind persistent storage. 
   It can function as a database, cache, and message broker. Redis on Flash, a Redis Enterprise feature, extends RAM with Flash memory 
   for cost-effective support of large databases. RDB (Redis database file) persistence takes point-in-time snapshots of 
   the data sets at specified intervals.
   
   Advantages: 
   Sub-millisecond latency and high throughput
   Widely supported among programming languages
   Free open source version has many features
   Enterprise version improves speed and eases clustering
   Enterprise version supports Redis on Flash
   Enterprise version implements Conflict-free Replicated Data Types (CRDTs)
   Modules support many data models and functionality extensions - this ensures clean , scalable design as requirement grow and stops it from becomming polyglot design.
   Redis Enterprise offers active-active deployment for globally distributed databases, enabling simultaneous 
      read and write operations on the same dataset across multiple geo-locations.
   Redis is available on AWS , GSP, Azure and standalone.

   Redis Sentinel, provides high availability for Redis. 
      It does monitoring of the master and replica instances, notification if there is something wrong, 
      and automatic **failover** if the master stops working. It also serves as a configuration provider for clients.
   
   2. Redis Streams :
      Redis streams are ideal for building history preserving message brokers, message queues, unified logs, and track systems. 
      Unlike Pub/Sub messages which are fire and forget, Redis streams preserve messages in perpetuity.
      In leader board's case Redis streams could be used to feed in the match elimination results , **scalable** and 
      **reliable** consumer groups will give ability to scale up down the streams consumption as loads vary.
      Redis stream also provides ability for clients to acknowledge the messages and store them when they are not acknowledged ,
      These capabilities provide ability to increase the **reliability** of the system in case of **failover**.
      Redis Streams provide the ability to add publishers and consumers using groovy , java etc.
   
   3. Rest APIs - We known spring boot and related libraries are good enough for leader board customer load which will be fetched from redis cache.
   
   4. RedisGears deployment for real time data processing:
      GearsCoordinator orchestrates the distributed execution of your functions on each shard in your database. 
      GearsExecuter schedules and triggers the execution of your functions. Functions can be triggered ad-hoc, by new entries in a stream, or by keyspace notification. In the latter, the function can be synchronously executed with the notification. (The GearsExecutor is not visible in the above diagram, but implied by the events/trigger section.)
      GearsEngine is the runtime execution environment of RedisGears.
      Leader board can use this capability to write **serverless logic** to trigger the functions operating on match results , adding scores to redis cache and saving incomming information to database.
      
  
      
    **Scalability , Reliability , resilience , Capacity Planning.**
    
    I have been mentioning scalability,Reliabality , resilience  in above data model choices selection , tech stack selection still would try to mention here.
    
    1. At the event processing level , 
        Redis streams , Redis Gears serverless are linearly scalable as demand grows. Different topologies of event consumption could be used and systems could be scaled up and down
        Reliability could be achieved using Redis streams acknowledge message funtionality , when consumers go down or have some propblems no event would be lost.
        As leader board will be running from redis cache which is synched with disk durin degradation at most the updates will start reflecting slowly and resilence will be shown.
        Capacity planning could be achieved with adding additional nodes to Redis clusters.
                
    2.  Rest API level  -  Spring Boot Microservices can scale at high levels specially when data for leaders board is fetched from redis cache and Microservices are running in cloud environment.
        Microservices deployment with proven even driven approach , blue green deplyments stratgeies ,  ensures required aspects.
         
    3.  Redis Key value data store and as and when  needed RedisSearch, RedisGraph, RedisJSON, RedisTimeSeries, and RedisAI etc highly scalable , reliable , resilent for the data operations.
        
     cost optimization: 
        Redis on flash saves RAM cost and allows disk:ram storage ratio to be changed as and when needed.
        Redis Multimodal database keeps future requirmenet for searches , graphs possible at the same time saves money for new databases.
      
      Data Loss:
        System gurantees 100% data processing gurantee , at application level time stamps at score level and events saved by redis event streams  allow failover without a data loss.
       
      Alerting and monitoring is taken care by Redis Sintel , as well as application level notifications could be implemented say after system starts , process certain events etc.
      This could be hooked on to incident mamnagement tools. 
      
      Blue green kind of deployment strategies should be able to improve on  downtime etc during changes in the system.
     
      
         
      
           
       
   
      
            
      
        
   
 
 
 
 
 
 
 
 
 
    
 
 
 
 
 
 
 
 