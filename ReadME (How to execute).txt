                                           ##################################################################################
                                           #                                                                                #
                                           #                     	  How to execute code?	                            #
                                           #                                                                                #
                                           ##################################################################################




							 _______________________________
							|   				|
							|    	STARTING HDFS		|
							|_______________________________|


1. Install Hadoop, Flume and Hive as directed in the "Code" folder.

*******************************************************************************************************************************************************

2. To check whether Hadoop is installed, type following code in terminal:

	$ hadoop version

*******************************************************************************************************************************************************

3. Start Hadoop dfs (distributed file system) and yarn using following command:

	$ start-dfs.sh
	
	$ start-yarn.sh


*******************************************************************************************************************************************************

4. Type the following command to check the running daemons in yarn:

	$ jps

*******************************************************************************************************************************************************

5. Create directory in HDFS using the command:

	$ hdfs dfs -mkdir hdfs://localhost:9000/major

*******************************************************************************************************************************************************





							 _______________________________
							|   				|
							|    CREATING A TWITTER API	|
							|_______________________________|



6. Twitter API requires creation of twitter app and generation of consumer keys and access tokens. The steps to create Twitter API are as follows:

	(a) You need to have a working Twitter account for the authentication purposes. Twitter doesn’t allow access to its data if one             doesn’t have a twitter developer account. So, we need to fill out the details regarding the purpose that we need the data               for.

	(b) Visit developer portal with and approved developer account to create a Twitter app and generate authentication keys.

	(c) These include:
		'Consumer’ tokens – for app authentication and,
		‘Access’ tokens – for user /account authentication.

*******************************************************************************************************************************************************





							 _______________________________
							|   				|
							|        CONFIGURE FLUME	|
							|_______________________________|



7. Configure Flume to copy data to HDFS

	(a) Create a file flume.conf inside /usr/lib /apache-flume-1.6.0bin/conf and add the following parameters into it:	

		# Naming the components on the current agent.
		TwitterAgent.sources = Twitter
		TwitterAgent.channels = MemChannel
		TwitterAgent.sinks = HDFS

		# Describing/Configuring the source
		TwitterAgent.sources.Twitter.type = org.apache.flume.source.twitter.TwitterSource
		TwitterAgent.sources.Twitter.consumerKey = Your consumer key
		TwitterAgent.sources.Twitter.consumerSecret = Your consumer secret key
		TwitterAgent.sources.Twitter.accessToken = Your consumer key access token
		TwitterAgent.sources.Twitter.accessTokenSecret = Your consumer key access token secret
		TwitterAgent.sources.Twitter.keywords = elections2019

		# Describing/Configuring the sink
		TwitterAgent.sinks.HDFS.type = hdfs
		TwitterAgent.sinks.HDFS.hdfs.path = hdfs://localhost:9000/user/Hadoop/twitter_data/
		TwitterAgent.sinks.HDFS.hdfs.fileType = DataStream
		TwitterAgent.sinks.HDFS.hdfs.writeFormat = Text
		TwitterAgent.sinks.HDFS.hdfs.batchSize = 1000
		TwitterAgent.sinks.HDFS.hdfs.rollSize = 0
		TwitterAgent.sinks.HDFS.hdfs.rollCount = 10000

		# Describing/Configuring the channel
		TwitterAgent.channels.MemChannel.type = memory
		TwitterAgent.channels.MemChannel.capacity = 10000
		TwitterAgent.channels.MemChannel.transactionCapacity
		= 100

		# Binding the source and sink to the channel
		TwitterAgent.sources.Twitter.channels = MemChannel
		TwitterAgent.sinks.HDFS.channel = MemChannel


	(b) Start Flume through following command:

		$ cd $FLUME_HOME
		$ bin/flume-ng agent –conf ./conf/ -f conf/flume.conf Dflume.root=DEBUG,console -n TwitterAgent


	(c) If everything goes right, tweets will start streaming into HDFS. Now we can verify this by accessing Hadoop Administration Web UI using the
	    URL:

		https://localhost:50070/

*******************************************************************************************************************************************************





							 _______________________________________
							|   					|
							|        ANALYSIS USING HIVE QUERIES	|
							|_______________________________________|



8. After successful installation of Hive, it can be verified by using the following command:

	$ cd $HIVE_HOME
	$ bin/hive


9. Now, we need to extract the data from HDFS (unstructured format) into our hive tables (structured format) for further analysis.

	 Create external table to load twiiter data into hive:
		
		CREATE EXTERNAL TABLE raw_f (
		   id BIGINT,
		   created_at STRING,
		   source STRING,
		   favorited BOOLEAN,
		   retweeted_status STRUCT<
		      text:STRING,
		      `user`:STRUCT<screen_name:STRING,name:STRING>, retweet_count:INT>,
		   entities STRUCT<
		      urls:ARRAY<STRUCT<expanded_url:STRING>>,
		      user_mentions:ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
		      hashtags:ARRAY<STRUCT<text:STRING>>>,
		   text STRING,
		   `user` STRUCT<
		      screen_name:STRING,
		      name:STRING,
		      friends_count:INT,
		      followers_count:INT,
		      statuses_count:INT,
		      verified:BOOLEAN,
		      utc_offset:INT,
		      time_zone:STRING>,
		   in_reply_to_screen_name STRING
		) 

		ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
		WITH SERDEPROPERTIES ("ignore.malformed.json" = "true")
		LOCATION 'hdfs://localhost:9000/major';

							TWITTER DATA ANALYSIS

	1. Finding Recent Trends and Most Popular hashtags

		hive> CREATE VIEW L5 AS SELECT
			ht,
			count(ht) AS countht
			FROM raw_f LATERAL VIEW
			EXPLODE(entities.hasgtags.text) dummy AS ht
			ORDER BY countht DESC limit 10;

	2. Finding Users with the greatest Number of Tweet

		(i) Clean up tweets

			CREATE VIEW simple_tweets AS SELECT
			  id,
			  `user`.screen_name,
			  source,
			  retweeted_status.retweet_count,
			  entities.hashtags,
			  cast ( from_unixtime( unix_timestamp(concat( '2016 ', substring(created_at,5,15)), 'yyyy MMM dd hh:mm:ss')) as timestamp) ts,
			  text,
			  `user`.statuses_count,
			  `user`.friends_count,
			  `user`.followers_count,
			  `user`.time_zone  
			FROM raw_f;		

		(ii) The query which will give us the user with maximum number of tweets is:

			hive> CREATE VIEW L7 AS SELECT
				screen_name,
				count(id) AS cnt
				FROM tweets_simple
				GROUP BY screen_name
				HAVING screen_name IS NOT NULL
				ORDER BY cnt DESC LIMIT 10;

	3. Performing sentiment analysis on the tweets

		(i) Removal of stopwords and cleaning

			hive> CREATE VIEW remove_stopwords AS SELECT * 
				FROM exploded_view 
				WHERE exploded_view.word NOT IN (SELECT * FROM stopwords) AND regexp_replace(exploded_view.word,'[^a-zA-Z0-9]+','') != '';

		(ii) Tokenizing the tweets

			hive> CREATE VIEW L1 AS SELECT
				id,
				words
				FROM raw_f LATERAL VIEW
				EXPLODE(sentences(lower(text))) dummy AS words;

			hive> CREATE VIEW L2 AS SELECT
				id,
				word
				FROM L1 LATERA VIEW
				EXPLODE(words) dummy AS word;

		(iii) Sentiment word detection

		* Creating external table for dictionary
	
			hive> CREATE EXTERNAL TABLE dictionary (
				type STRING,
				length INT,
				word STRING,
				pos STRING,
				polarity STRING
				)

				ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS
				TEXTFILE;

		* Converting polarity into numeric value

			hive> CREATE VIEW L3 AS SELECT
				id,
				L2.word,
				CASE d.polarity
				WHEN ‘negative’ THEN 0
				WHEN ‘positive’ THEN 5
				ELSE 2.5 END AS polarity
				FROM L2 LEFT OUTER JOIN dictionary d on
				L2.word = d.word;

		* Classification of words into positive, negative or neutral

			hive> CREATE
				id,
				CASE
				WHEN
				WHEN
				ELSE
				FROM
				TABLE tweets_sentiment AS SELECT
				AVG (polarity) > 2.5 THEN ‘positive’
				AVG (polarity) < 2.5 THEN ‘negative’
				‘neutral’ END AS sentiment
				L3 GROUP BY id;

		(iv) Classification of tweets

			hive> CREATE VIEW L8 AS SELECT
				sentiment,
				COUNT (id)
				FROM tweets_sentiment AS ts
				GROUP BY sentiment;

	4. Remove SPAM

		CREATE VIEW remove_spam AS
		  SELECT * 
		  FROM simple_tweets 
		  WHERE 
		   statuses_count > 50 AND
		   friends_count/followers_count > 0.01 AND
		   length(text) > 10 AND
		   size(hashtags) < 10;









