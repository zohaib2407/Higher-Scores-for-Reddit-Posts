# Higher-Scores-for-Reddit-Posts
This project aims to understand the factors that influence the “score” (=upvote – downvote1 value) of a post

## PROBLEM SETTING

The (anonymous-by-choice) social media platform, Reddit, has a much smaller user base (430 million active users against Facebook’s nearly 3 billion). Reddit allows its users to create “posts” (text, image, link or video based) in “subreddits” specific to certain topics (decided by the community of users “subscribed” to that community). Interactions on these posts (through discussion in the comments and through up/down-voting) foster its user base.

This project aims to understand the factors that influence the “score” (=upvote – downvote  value) of a post, aside from the post’s specific content itself, as it has been observed that posts with nearly identical content get different scores based on factors such as the subreddit it was posted in, the time (w.r.t GMT) it was posted in, the user that posted it, the type of content posted, among multiple others.

## DATA DESCRIPTION

Reddit makes almost all of its data available via its APIs, and there are a few services that allow real-time extraction of data through them. However, since this project’s focus is on static data as opposed to real-time analysis of activity on Reddit, we rely on Pushshift’s monthly dumps of Reddit’s content on its servers. Pushshift collects Reddit submissions, comments, moderator, and user information among others. 

The data has been extracted from a service that hosts monthly dumps of Reddit’s data in various repositories – this project focuses on the “submissions” data i.e. posts, specifically for the month of 2021-06, the latest available data on the service.
The is stored using Facebook’s ZSTD compression algorithm, and is in the form of JSON files, with varying sets of information available per post. The specific dataset available for 2021-06 is approximately 8.81 GB when compressed. The plan for the project is same as that submitted in the project plan.

The details of cutting down on data that is necessary for the objectives of the project is detailed below:
1.	The dataset can be extracted using Facebook’s ZSTD binary provided for Windows, specifically after providing the flag –long=31, and will result in a 125GB JSON file.
2.	This JSON has been processed in Python by loading the file line by line, and using Python’s built-in package, JSON, the JSON members that are common to all records is determined (along with all keys available).
3.	Out of 124 JSON members, 77 of them are present across all the 34,118,481 records present in the file.
4.	These are extracted into a CSV file, using various tricks to reduce the size of the file:
a.	Records where the original post has been deleted are marked as being under [deleted] user and the contents of which are [deleted], which render it unfit for modelling. They are re-coded to blanks.
b.	When no information is available on some specific JSON members that are encoded as lists, they are marked with {} or [], and are re-coded to blanks. This is also done when the datatype is NoneType.
5.	The final size of this file is approximately 26GB in size.


## SETTING UP THE PYSPARK INFRASTRUCTURE

Google offers a free trial with $300 in credits for their Google Cloud Platform. This was leveraged to setup a system with 32GB of memory and 6 vCPUs on an Ubuntu system, on which Spark was installed. We built the entire infrastructure on the remote virtual machine to have higher computation power and efficient operations when working with BIG data. The set-up process included installing miniconda, python, spark, jupyter, and other dependencies in addition to transferring all the data and programs from our local system to remote machine using SCP.  With the remote virtual machine running, we run our Jupyter installation at port 8888, and then using an SSH client we tunnel that port to our local browser at some arbitrary port (say 8886). Team members now work in their local browser but all the computations will be made on the remote virtual machine.  

## Data Extraction and Data Loading 

The dataset, at 26GB is still too large to subject to ML algorithms for most techniques. The following additional filters were applied, therefore, to make the dataset suitable for analysis, meaningful (in that it captures the patterns in the data better), and applicable to the most “popular” subreddits that are likely generalizable to the average post on the platform:

1. The top 20% of subreddits in terms of subscriber count (≥353,518 subscribers) are retained.
2. Active subreddits i.e. have at least 8 posts per month are retained (the cutoff for top 20% of posts/month).
3. Submissions i.e. with a score of at least 2 i.e. upvoted by 2 users other than the original poster (OP ) are retained.

This covers approximately 1,400 subreddits, and around 3.4 million data points.

It is difficult to extract a 125GB file compressed to 8GB via standard tools like pandas in Python. That is why we had to process the data line-by-line using a Python file iterator and use its JSON parser by line to identify the common members that could later be extracted on a second run. The critical element of this analysis would be meeting the memory requirement.

![image](https://user-images.githubusercontent.com/35283246/163795852-1d47d344-d017-43d6-9b21-8d40ca28887c.png)

Reading the data from a very large CSV file will be incredibly slow, especially for a system based off on a hard disk drive. We convert the data format to parquet, which loads around a thousand times faster.

Parquet is an open-source file format built to handle flat columnar storage data formats. Parquet operates well with complex data in large volumes. It is known for its both performant data compression and its ability to handle a wide variety of encoding types. Feature analysis is done using Parquet.
 
Some Parquet benefits include:
1. Fast queries that can fetch specific column values without reading full row data
2. Highly efficient column-wise compression

## Data Cleaning and Exploration

We have sued different methodologies to clean the data and bring it into a concise format for further exploration and modelling. Only the columns of interest to determine the influence on score have been included in the modelling. Casting has been used to convert the various columns to valid formats. When posts are deleted on Reddit, their author names are marked under "[deleted]". This will be construed as a common poster across almost every subreddit, which we want to avoid. While coding the data from the JSON file to CSV, we coded these as null values.
This leads to us arriving at 28 million records. Let's choose subreddits with at least 353,518 subscribers - that is, around the top 20% of all subreddits in the data.

We have then checked to validate the quantile score and further filtered for the minimum subscriber count. This brings the record count to under 6 million, which is a significant amount. Next, we checked the number of non-null columns in the data and change the columns that are to some other value.

The columns with limited number of non-null values making them less useful for modelling have been dropped. The next step is to start cleaning columns. The data types of certain columns have been changed for further cleaning the data.

1.	Score primarily varies from 3 to around 230,000, with over 90% of the data being under 371.
2.	Subreddit subscribes are significantly higher in magnitude, with a median value of 91,000.
3.	Crossposts are largely 0, even to the 99%ile range.
4.	Comment interactions are above 4 only beyond the 50% mark.
5.	Awards received is strongly tied to the score of the post (higher scores get higher awards, especially at the top, and are likely cases of data leakage.
6.	The ads_ui flag is practically irrelevant.
7.	All posts have the thumbnail flag as True, so it is also not helpful in getting noticed.
8.	Almost 80% of posts have some form of Media Embedding.
9.	The title length averages at 38 characters (in terms of median).

The total_awards_recieved is, as observed earlier, highly correlated (0.56) to the score, and is a clear case of data leakage.score and num_crossposts are also correlated (0.56), however, they drive each other, and is not as clear a case of data leakage and can thus still be included in the model.score is loosely tied to num_comments (0.13) and has a similar reasoning as num_crossposts.There is a general negative correlation all "positive" measures of popularity (subscriber count, is_video, is_self, title_length etc.) with the adult-content flag, over_18, but does not have any linear relationship to score.

Adding additional time-based measures<br>
Functions are designed to get specific periods such as the weekday, or hour of the data into a new column from the datetimefield.

![image](https://user-images.githubusercontent.com/35283246/163796366-b4270f20-e09e-4674-8107-cc739db216ac.png) ![image](https://user-images.githubusercontent.com/35283246/163796387-8f6562e5-5a34-4419-9a48-6cd3338c02f4.png)

![image](https://user-images.githubusercontent.com/35283246/163796406-81b5d35c-b649-49e1-aa0e-5eb1d4f269cc.png)
While it may be easy to conclude that mean score increases by title length, there are far fewer titles with very high title length. To avoid misleading images, a plot for title_length values less than 100 is in order.

![image](https://user-images.githubusercontent.com/35283246/163796431-b0f8267a-d814-447e-8211-98b5622450e2.png)
As can be seen, this view paints a different picture - the score is higher on average for lower title counts.

From the above analyses, treating score as a categorical variable will not yield satisfactory results - the score itself varies widely, with a median of 22, and a range between 3 and 230,000. What can then be done is to create a model for the data that crosses a certain threshold - let's say, the top 50% of posts, which have a score of 22, and then bucket the score into specific groups.

Based on the values above, the bins for the score can be created. The final output is saved to disk to reload as PySpark DataFrame.

## Modelling Using spark.MLib

#### Preparing Data for Ingestion
Spark's MLib is absolutely sparse on documentation. It cannot handle string data, and so strings have to be "transformed" into numeric values. Reference for the following steps:

**StringIndexer**<br>
PySpark has a feature called StringIndexer, which encodes the string into numeric values. The reverse can also be performed via IndexToString.

**VectorAssembler**<br>
For some godforsaken reason, PySpark also wants the features to be in a single column, stored as a vector of features in each individual row. This is achieved through the following code:











