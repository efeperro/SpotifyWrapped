# Technical Report: Spotify Wrapped - A Music Streaming Analysis

## Introduction
Spotify Wrapped has become a tradition for music listeners within the Spotify platform. The data that Spotify manipulates, however, is collected in the streaming context for analysis. Stream analytics offers many tools and approaches towards analyzing data in the streaming context, and knowledge of components that aid in the processing of it. This project focuses on analyzing the data generated after completing AVRO serialization of a simulated Spotify record schema. The purpose of this analysis is to understand user behaviors and preferences. These insights can inform decisions in areas such as marketing, product development, and customer engagement, simulating the Spotify Wrapped event. This [Link](https://github.com/efeperro/SpotifyWrapped/tree/main) will redirect you to the repository for this project.

 ## **Table of contents:**
 - ### [Schema Definition](#item-one)
 - ### [Design of Synthetic Data Generation](#item-two)
 - ### [Setup & Code Design](#item-three)
 - ### [AVRO Producer File Generation](#item-six)
  - ### [AVRO consumer File Generation](#item-seven)
 - ### [Implementation of Query Analyses](#item-four)
 - ### [Challenges Encountered](#item-five)

 

## Objectives

* Select the three most insightful analyses related to band or user data handling to draw conclusions from a Spotify Wrapped data feed.

* These analyses attempt to leverage streaming analytics concepts, such as the use of different window types in analyzing real time data, notifications to Azure queues...
* Implement the technical workflow where AVRO records are produced in Google Colab and sent to a Kafka topic. From Kafka, Spark is utilized to consume the records and conduct the three analyses chosen by the team.

## <a id="item-one">Schema Definition</a>

* After generating an AVRO file schema, it was concluded that the following fields are essential for properly analyzing the data generated within the AVRO record producer. You can see more information about the Kafka record producer **here**.

### Schema:


| Field                | Data Type                  | Description                                                                                  |
|----------------------|----------------------------|----------------------------------------------------------------------------------------------|
| UserId               | string                     | Unique identifier for the user                                                               |
| Age                  | string                     | Age of the user                                                                               |
| Gender               | string                     | Gender of the user                                                                            |
| ListeningDevice      | string                     | Device used for listening to music                                                            |
| SubscriptionPlan     | string                     | Subscription plan of the user                                                                 |
| MusicTimeSlot        | string                     | Time slot when the music was listened to                                                       |
| Location             | string                     | Location of the user                                                                          |
| SongId               | string                     | Unique identifier for the song                                                                |
| TrackName            | string                     | Name of the track                                                                             |
| Artist               | string                     | Name of the artist                                                                            |
| Genre                | string                     | Genre of the song                                                                             |
| SongStart            | string                     | Timestamp indicating when the song started playing                                             |
| SongEnd              | string                     | Timestamp indicating when the song ended playing                                               |
| Length               | int                        | Length of the song in seconds                                                                 |
| InteractionType      | enum: InteractionTypeEnum | Type of interaction with the song (PLAY, PAUSE, SKIP, LIKE, ADDED_TO_PLAYLIST)                  |
| InteractionTimestamp | string                     | Timestamp indicating when the interaction with the song occurred                                |

### Considerations:

This schema definition differs from the initial version regarding record fields, and the available values. Initially, `SongEnd` and `SongStart` are new record fields that allow to calculate more realistic timestamp for the listening sessions per user. furthermore, all of the record fields are required to contain a record value with the proper data format. The previous schema allowed nullable values and allowed many errors to be inputted, so this fix ensures that the proper values are being inserted (and all record values are generated given the synthetic data's fully available nature). Lastly, this schema was utilized for the realistic data generation to be fed.

## <a id="item-two">Design of Synthetic Data Generation</a>

### Realistic Distribution Formats

To align to the realistic data generation objective, real spotify data was extracted and wrangled from Kaggle to adjust to real distributions of spotify users. After careful consideration, it was concluded that only 2 datasets were required. The first dataset, `user_data`, contains real values of user's demographics and personal information, and our second dataset `music_data` contains the metadata that is needed to filter the designated interactions of a user. This dataset is used as a json file, and converted to a dictionary file for the proper imputation to the AVRO schema. These datasets can be accessed within the directory.

After loading our data, distributions were extracted for field imputation:

* The record fields `gender`, `age`, `subscription plans`, and `listening device` are considered to be fields unrelated to a user's preference or personality. For this reason, the approach of data generation is to be extracted from a distribution based on options. For example, the count of all record values for these fields were peformed in order to extract all possible options for the fields, and then joining those options as a list with the normalized probability distribution. This way, all of the realistic demogrpahic possibilities are taken into account, while also ensuring a realistic filtering decision based on the most probable field imputation. For better context, refer to Table 2 below, which depicts the probabilities of the users' real distribution of subscription plans.


| Subscription Plan                | Probability           |
|----------------------------------|-----------------------|
| Free (ad-supported)              | 0.8153846153846154    |
| Premium (paid subscription)      | 0.18461538461538463   |

Table 2: probabilities of the user's real distribution of subscription plans


### Data Classes for Listening Personalities

The implementation of data classes completely improved the data generation script given the previous random choice of music approach that dismissed the real user interaction that involves listening preferences, mostly in the genre context. The Listening Personality data class defines 8 different personalities with listening preferences of 2-3 preferred genre, and added probabilities of active hours (listening time-slots) and interaction probabilities (likes, skips, add to playlist). The last probability definition, translates the number of records per each user based on the active hours that each user is (a user wiith 10 active hours would have more records than another user with 5 active hours). After the definition, the synthetic data generation randomly chooses a personality type, and filters the available genres from the selected personality. from this checkpoint, the available genres are filtered by the preferred genre in the record and extracts a song from it. On another record, the genre choice might change depending on the user's preferred genre.

Here's a table describing each listening personality:

| Listening Personality   | Name               | Genre Distribution                      | Active Hours   | Interactions                              | Number of records (per user) Probability |
|-------------------------|--------------------|------------------------------------------|----------------|-------------------------------------------|-------------------------|
| Rap Rock Jazz Lover     | RapRockJazzLover  | hip hop: 0.6, rock: 0.3, jazz: 0.1      | 17:00 - 23:00  | skip probability: 0.1, like probability: 0.3, playlist add probability: 0.2 | 0.3                     |
| Indie Soul Fan          | IndieSoulFan       | indie: 0.5, soul: 0.5                   | 09:00 - 17:00  | skip probability: 0.15, like probability: 0.25, playlist add probability: 0.1 | 0.7                     |
| Electronic Explorer    | ElectronicExplorer| electronic: 0.7, house: 0.2, techno: 0.1 | 22:00 - 04:00  | skip probability: 0.05, like probability: 0.2, playlist add probability: 0.25 | 0.5                     |
| Pop Punk Person         | PopPunkPerson      | pop: 0.4, punk: 0.6                     | 15:00 - 22:00  | skip probability: 0.2, like probability: 0.5, playlist add probability: 0.3 | 0.8                     |
| Classical Connoisseur  | ClassicalConnoisseur| classical: 1.0                          | 08:00 - 20:00  | skip probability: 0.05, like probability: 0.4, playlist add probability: 0.15 | 0.9                     |
| Jazz Junkie             | JazzJunkie         | jazz: 0.9, blues: 0.1                   | 18:00 - 24:00  | skip probability: 0.1, like probability: 0.35, playlist add probability: 0.25 | 0.3                     |
| Country Cruiser         | CountryCruiser     | country: 0.8, folk: 0.2                 | 10:00 - 18:00  | skip probability: 0.12, like probability: 0.3, playlist add probability: 0.18 | 0.4                     |
| Reggae Relaxer          | ReggaeRelaxer      | reggae: 0.7, dancehall: 0.3             | 16:00 - 23:00  | skip probability: 0.08, like probability: 0.4, playlist add probability: 0.22 | 0.6                     |

I made the adjustments by removing underscores, removing quotation marks, adding spaces to words without spaces, and capitalizing the first letter of each word.

Table 3: summary of each listening personality, including their name, genre distribution, active hours, and interaction probabilities.


## <a id="item-three">Setup & Code Design</a>

The ipynb file uploaded to the repository initializes with the installation of module packages necessary such as:


* `fastavro` for schema parsing and data serialization.
* `Faker` for assistance in synthetic data generation.
* `pandas` and `numpy` for data wrangling and structurization.
* `timezonefinder` to properly define timestamp formats.
* `kafka-python` to call Kafka into python language integration.
* `pyspark` to create a Spark session and connect the event hub to Kafka.

This section initializes the environment by installing necessary libraries and tools. This section sets the foundation for subsequent data handling and streaming tasks.

**Kafka**: the Kafka configuration simply introduces Apache Kafka, with its version along with configuration definition; ensuring all components are ready for data streaming. This includes scripts to download, install, and configure Kafka on the host system. Finally, the last step performed in the setup is the Kafka topic definition (called spotifyWrapped) to which the data will be generated (for AVRO producer and consumer to be defined within the topic), as well as the partitions and replication factor definition. For such a low volume task, the final decision defined 3 partitions, with a replication factor of 1.

### <a id="item-six">Avro Producer File Generation</a>


The AVRO producer file is generated within the notebook. The producer record file inlcludes every step of the synthetic data generation. Initially, the AVRO schema is defined to properly understand the record imputation from the function along with the complementary code such as the data classes and distribution formats. The approacch however, differs slightly within the record generation by implementing a for loop for a range of 10,000 iterations. This means that 10,000 users will create records, and another for loop defines how many records this person will generate depending on the interaction probability. Therefore, a user can generate 50 records at most. The reason for this approach occurs since the analysis queries require constant record generation from the producer to grasp and query the data. Then, after each record is generated, it is then serialized and sent to the Kafka topic as a message. A new message is generated approximately each second. 

### <a id="item-seven">Avro Consumer File Generation</a>


the AVRO consumer simply outlines the schema once again to certify the AVRO producer's messages, and be able to deserialize the data records being sent to the topic. Then, after deserialization, the records that are received are consumed by the file. 

### Spark Setup & Connection

The Spark setup maintains an approach similar to Kafka, installing the necessary packages for the integration of the data from the Kafka tasks. Once again, we are required to define the schema from the Spark, and then start a Spark session in order to be able to call the data stream. this Spark session finally allows to read the event hub utilizing Kafka. From this checkpoint, the analyses can be implemented. This section details the setup for integrating Spark into the data processing workflow, enabling robust data analytics capabilities.


## <a id="item-four">Implementation of Query Analyses</a>

From the generation of 10 different queries, only 3 query analysis were chosen. The final analysis chose surges from the insightful content and the relation of the insight with the real context of a Spotify Wrapped output data. the queries process and analyze the data to generate actionable insights. This functionality focuses on using the results from the queries to understand user behavior and to identify potential areas for improvement.

### Query Implementations

1. **Artist Engagement Metrics**: This query calculates engagement metrics for both artists and users. For artists, it aggregates the total number of plays, likes, and skips for each artist. For users, it aggregates the total number of plays, skips, and likes for each user.

-  This query specifically demonstrates the total plays and likes, giving the artist an idea of the impact on users. This is a very important query due to the real analysis that artists are given regarding their listening sessions from the users interested in their music.

```python
artist_popularity_query = df \
    .groupBy(col("Artist")) \
    .agg(
        sql_sum(when(col("InteractionType") == "PLAY", 1).otherwise(0)).alias("Total Plays"),
        sql_sum(when(col("InteractionType") == "LIKE", 1).otherwise(0)).alias("Total Likes")
    ) \
    .writeStream \
    .outputMode("complete") \
    .format("memory") \
    .queryName("artist_popularity") \
    .start()
```


2. **User Engagement**: This query focuses specifically on user engagement metrics. Once again, it is important to give emphasis to both users and artists. In this specific case, the user-emphasized query calculates the total number of plays, skips, and likes for each user, providing insights into user behavior and preferences.

```python
engagement_query = df \
    .groupBy(col("UserId")) \
    .agg(
        sql_sum(when(col("InteractionType") == "PLAY", 1).otherwise(0)).alias("Plays"),
        sql_sum(when(col("InteractionType") == "SKIP", 1).otherwise(0)).alias("Skips"),
        sql_sum(when(col("InteractionType") == "LIKE", 1).otherwise(0)).alias("Likes")
    ) \
    .writeStream \
    .outputMode("complete") \
    .format("memory") \
    .queryName("user_engagement") \
    .start()
```


3. **Identifying Top Artists by Region**: This last chosen query takes an emphasis towards Spotify's attempt to know more about both the user and artist as context. The query identifies the top artists by region, showing the popularity of artists based on the number of listeners in each region. This query is highly important not only for an artist, but to the company itself to aid in the service of the user. Many generation of playlists from Spotify are performed to increase the users' engagement, and a very common approach from Spotify is to create playlists based off users' main interests by location.

```python
artist_region_popularity_query = df \
    .groupBy("Location", "Artist") \
    .agg(
        count("UserId").alias("Listener_Count")
    ) \
    .writeStream \
    .outputMode("complete") \
    .format("memory") \
    .queryName("artist_region_popularity") \
    .start()
```


#### Other Considered Queries: 
Given that Spotify is a streaming service company must create numerous analytics to better undertand the users, other queries offer really good insights. The reason these were not implemented are simply the number of queries aas a constraint, and the power of the 3 previous chosen queries. Another constraint is the unavailability to generating queries grouped by window sessions, which is further discussed in this **section**. Still, 4 of these are worth to mention:

1. Top Genres by Listening Duration: This query calculates the total listening duration for each genre within defined time windows. It provides insights into the popularity of different genres based on the total listening time per window.

2. Genre Popularity by Location: This query analyzes the popularity of genres in different locations. It aggregates the counts of each genre in various locations, highlighting regional preferences in music genres.

3. Likes Count by Song, Artist, and Location per Window: This query focuses on counting likes for songs by considering factors such as the song, artist, and location within specified time windows. It helps understand user preferences and popularity trends for songs and artists in different locations over time.

4. Listening Habits Analysis: This query examines users' listening habits by grouping interactions based on the time slot of the day. It provides insights into when users are most active in terms of music listening, aiding in the optimization of content delivery and personalized recommendations.


### Query Enhancements for Realistic Simulation

#### Calculation of a User's Real Listening Time: 
Understanding a user's proper listening time, the listening time timestamps should be realistic. This means that if a user interacted with a SKIP interaction, then it is impossible for the track length to be the complete length of the song. To simulate this, if a user skips the song, then the length of the song will be multiplied by a random probability generator. This will ensure that the skip interaction reduces the listening time, and can be added up to the total listening time per user. To accurately calculate the amount of time a user listened during a session, additional records such as SongStart and SongEnd were necessary. This allowed for the inclusion of listening duration in data groupings. Then, the record interaction can be analyzed through listening time by substracting EndTime timestamp by the StartTime timestamp; yielding the real time of the interaction.


## <a id="item-five">Challenges Encountered</a>

### Data Generation Realism
The first challenges that appeared was distributing genre preferences per user without filtering the number of genres from the dataset, which required careful handling to maintain realistic user profiles. The context of a user's real preferences and interests when it comes to listening to music was not being properly assesed since the records were generated from random data extractions, and failed to show a human pattern regarding listening personalities. Therefore, a lot of time was given to the improvement of the synthetic data generation. In other words, proper analytics cannot be succesfully performed with insightful results without realistic data. Still, this challenge was tackled initially by the implementation of data classes and real data distributions. 

### Schema Management: 
The data types for selected record fields of the AVRO schema involved thoughtful consideration. Different formats of timestamps presented difficulties during query execution, complicating the extraction and comparison of time-based data. The reason for this is the timestamp formats for numerous fields, that had many collisions as some had a different format from others. In order to fix this, the timestamps were defined as string data types, and then converted to timestamp data types at the moment of extracting such fields for data conversion and querying.

### Consistency in User Data: 
At some point in the data generation, issues regarding the user id and location surged as these fields did not have consistency for the same single user. As these did not match at all, it hindered the analysis per user as well as the integrity of the streaming data. However, the hassle of this challenge proved to be more than its fix, given that the definition of this variable was performed within a function that would iterate over the fields. this was quickly then fixed by making field definitions of location and user id before the record iterations for this single user.

### Real-time Data Processing: 
Dealing with the parameters of the constancy of record generation and volume of data in a streaming context posed challenges that were the most challenging to solve. The major challenge involves understanding the speed towards each record generation, and also making sure that the analyses have time to properly extract the records being produced to output the query. This was partially solved on some queries (only queries without time windows) since the waiting time for the query to terminate had to be adjusted.

### Window Time Queries:
After ensuring the consistency and integrity of both the data generation and the AVRO producer and consumer files, the window time queries correctly ran without querying the data being generated. the main assumption towards this error is the session timestamps that collide with the parameters of the window, sliding intervals, and the implementation of watermarks on selected queries. This means that the query should either be given more time to process the query, or the listening session timestamp should be edited to fit a window time query. This is the only challenge that could not be corrected, but given the assumptions, this will be soon fixed.

## Conclusion

The project successfully achieves its objectives by designing and executing queries that provide meaningful insights into user behavior and preferences. Three selected queries focus on artist engagement metrics, user engagement, and identifying top artists by region. These queries offer valuable insights into user engagement with artists, user behavior, and artist popularity across different regions. Additionally, the project addresses challenges encountered during implementation, such as ensuring data generation realism, managing schema consistency, and handling real-time data processing.

Despite some challenges faced during implementation, the project demonstrates the feasibility and importance of stream analytics in extracting valuable insights from streaming data. By leveraging stream analytics techniques and synthetic data generation, the project provides a foundation for understanding user behavior and preferences within the Spotify platform, ultimately contributing to the enhancement of user experience and platform performance.
