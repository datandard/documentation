## Exports

We store data in a very raw format, so a lot of processing is needed to export news articles, including:

 - Normalizing tags, so replaced tags are not included. 
 - Articles should include language taken from the feed so they can be filtered easily.
 - Tags should be collapsed into an array when articles are joined with tags. Because they could be normalized through the first steps in this list, we should ensure uniqueness. 
 - We can have several urls from different feeds, but only a single url should be counted as an article. By removing other feeds we should collapse them into a single item. In that process, we should take the earliest published time stamps, join the tags and so forth.

This is done in a several of view, ending in a single view. Because This view is expensive to build, we will materialize at some point. All views are prefixed with `view_`.

### Tag normalizations

```sql
CREATE OR REPLACE VIEW view_tag_normalization AS 
  SELECT 
    t1.id as id, COALESCE(t2.term, t1.term) as term 
  FROM news_tag t1 
    LEFT JOIN news_tag t2 ON t1.replaced_by_id = t2.id
```

This creates a table with duplicate tags but keeping the original ids:

```sql
datandard=> SELECT * FROM view_tag_normalization LIMIT 5;
   id   |      term      
--------+----------------
 157805 | GDPR
 257028 | GDPR
 760297 | Eisenindustrie
 760298 | Elektrochemie
 760299 | Kathode
(5 rows)
```

In this case, the one `GDPR` tag used to be `gdpr`. 

## Fixing garbage in published / released timestamps
From time to time, the published timestamp is not set in the source, or it will be parsed into something garbage. We keep a sourced timestamp from when it was first discovered in a **feed**. As the articles are removed from feeds in a fairly short manner, it is a good estimate that the original published time stamp can be too far from the discovered. Also, if we have multiple time stamps for the same article (but discovered multiple times), we take the first timestamp. The cases are:

 - The published timestamp is max 10 days earlier than the created_by timestamp. Use this.
 - The published timestamp is later than the created_by (should not be possible), or earlier than 10 days or not present. Used created_by.

This does not fix the issue with the same article from multiple feeds. We use `MIN` on the valid timestamps as you can see earlier. 

```sql
SELECT 
 CASE
  WHEN NOW() < to_timestamp(published) THEN created_at
  WHEN CAST(EXTRACT(EPOCH FROM (created_at - to_timestamp(published))) AS bigint) < 864000 THEN to_timestamp(published)
  ELSE created_at
 END AS published
FROM news_article 
;
```
