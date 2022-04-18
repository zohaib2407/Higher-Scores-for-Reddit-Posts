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





