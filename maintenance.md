## maintenance of feeds

The system will find article specific feeds that will forver only contain a single article. They do not take long to extract and if we get throttled it will unlikely be an issus, but the system will be more effective if they are disabled. This can be done with the following query:

```sql
UPDATE 
  news_feed set active = 'f' 
WHERE id IN (SELECT feed_id 
                FROM news_article 
                GROUP BY feed_id 
                HAVING count(1) = 1
             ) AND created_at < NOW() - interval '2 days';

UPDATE 9407
```

This will disable feeds that has only had a single article and have been in the system for at least 2 days. 

## ... of articles
There is a special field in the `news_article` table called `attribution_status` that should be `NULL` at any time where you are not doing any attribution or maintenance. This can be set before running a script that changes core elements. For example, adding new scraped values for news articles, we first implemented at deployed the new attribution. Just after, we updated all articles to have the field with status 'planned'. Then our manual attribution script would simply go through that. 
