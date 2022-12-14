## Exports

We store data in a very raw format, so a lot of processing is needed to export news articles, including:

 - Normalizing tags, so replaced tags are not included. 
 - Articles should include language taken from the feed so they can be filtered easily.
 - Tags should be collapsed into an array when articles are joined with tags. Because they could be normalized through the first steps in this list, we should ensure uniqueness. 
 - We can have several urls from different feeds, but only a single url should be counted as an article. By removing other feeds we should collapse them into a single item. In that process, we should take the earliest published time stamps, join the tags and so forth.

This is done in a several of view, ending in a single view. Because This view is expensive to build, we will materialize at some point. All views are prefixed with `view_`.

### Tag normalizations
Is done in the view `view_tag_normalization`. See [Tag Normalization Documentation](tags.md) for more information.

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

## Selecting the articles
To make this look simpler, we create a view that is the cleaned up version of each article for each feed:

```sql
CREATE TYPE article_content AS (
 context text,
 contenttype varchar(500),
 language varchar(500),
 base text,
 value text
);
CREATE OR REPLACE VIEW view_articles AS
     SELECT 
      a.id,
      a.link,
      a.medias,
      CASE
       WHEN NOW() < to_timestamp(a.published) THEN a.created_at
       WHEN CAST(EXTRACT(EPOCH FROM (a.created_at - to_timestamp(a.published))) AS bigint) < 864000 THEN to_timestamp(a.published)
       ELSE a.created_at
      END AS published,
     f.language,
     ARRAY_AGG(distinct t.term) as tags,
     ARRAY_AGG(distinct w.name) as authors,
     ARRAY_AGG(distinct (
       c.context,
       c.contenttype,
       c.language,
       c.base,
       c.value
     )::article_content) as content
     FROM news_article a
      INNER JOIN news_feed f ON (a.feed_id = f.id)
      LEFT JOIN news_article_tags at ON (at.article_id = a.id)
      LEFT JOIN view_tag_normalization t ON (t.id = at.tag_id)
      LEFT JOIN news_author w ON (w.article_id = a.id)
      LEFT JOIN news_articlecontent c ON (c.article_id = a.id)
     GROUP BY 
      a.id, a.link, a.medias, f.language
;
```

## Running a query with filters
In most cases you want to limit the time stamp of the articles, and you want to select only those from a given tag set. Assuming the tag set to be `(1,2,3)` and a date range, you can select with the following query. 

```sql

```
