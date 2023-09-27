# LiveIndexingBatchSize

## Summary

This guide discusses properly configuring the live index batch size and why you would want to adjust it for performance reasons.

### What is live index batching?

LiveIndexingBatchSize means that after `X` number of posts, it will index those posts into Elasticsearch to make them searchable via Mattermost. By default, LiveIndexing is set to `1`, meaning an index job is run after every post. 

### Why would you want to change this?

On extremely active/busy servers having the index run after every post can cause a large volume of traffic to Elasticsearch, potentially exhausting the connections and hogging resources.

### What impacts does changing this value have?

The primary impact is that a post will be indexed into Elasticsearch after a threshold of posts is met and made searchable within Mattermost. In the configure guide below, we suggest setting this to the average # of posts every 10-20 seconds. So, if you make a post, you cannot find it via search for 10-20 seconds, on average. Realistically, no users should see or feel this impact due to the limited amount of users who are actively **searching** for a post this fast. You can set this to a lower average or higher average as well. Depending on your Elasticsearch server specs. 

During busy periods, this delay will be faster as more traffic is happening, causing more posts and a quicker time to hit the index number. During slow times, expect the reverse. 


## How to configure

1. You must understand how many posts per 10s your server makes. Run the below query and get your server's average posts per second.

    Note that this query can be heavy, so it's encouraged to run it during non-peak hours. 
    Additionally, you can uncomment the `WHERE` clause to see the posts per second over the last year. `31536000000` represents milliseconds in a year. 

  ```sql
    SELECT
      AVG(postsPerSecond) as averagePostsPerSecond
    FROM (
      SELECT 
        count(*) as postsPerSecond, 
        date_trunc('second', to_timestamp(createat/1000))
      FROM posts
      -- 	WHERE createAt > ( (extract(epoch from now()) * 1000 )  - 31536000000)
      GROUP BY date_trunc('second', to_timestamp(createat/1000))
    ) as ppd;
  ```

2. Decide the acceptable index window for your environment and multiply your average posts per second by that. We suggest 10-20 seconds. So, assuming you have 10 posts per second on average, you would do `10 posts per second * 20 seconds` to come to the number `200`. After 200 posts Mattermost will run an indexing job, so on average, there would be a 20-second delay in searchability.

3. Edit the config.json or run mmctl to modify the `LiveIndexingBatchSize` setting
  
    **Editing in the config.json**

    ```json
    {
      "ElasticsearchSettings": {
        "LiveIndexingBatchSize": 200
      }
    }
    ```

    **Editing via mmctl**

    `mmctl config set ElasticsearchSettings.LiveIndexingBatchSize 200`

    ** Editing via Env Var**

    `MM_ELASTICSEARCHSETTINGS_LIVEINDEXINGBATCHSIZE = 200`

4. Restart Mattermost