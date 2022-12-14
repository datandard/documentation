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
CREATE OR REPLACE VIEW view_articles AS
     SELECT 
      a.id,
      a.uuid,
      a.link,
      a.medias,
      CASE
       WHEN NOW() < to_timestamp(a.published) THEN a.created_at
       WHEN CAST(EXTRACT(EPOCH FROM (a.created_at - to_timestamp(a.published))) AS bigint) < 864000 THEN to_timestamp(a.published)
       ELSE a.created_at
      END AS published,
     f.id as feed_id,
     f.uuid as feed_uuid,
     f.language as feed_language,
     p.id as publisher_id,
     p.uuid as publisher_uuid,
     a.language as language,
     json_agg(distinct jsonb_build_object(
       'id', t.id,
       'term', t.term
     )) FILTER (WHERE t.id IS NOT NULL) as tags,
     json_agg(distinct jsonb_build_object(
      'name', w.name
     )) FILTER (WHERE w.name IS NOT NULL) as authors,
     json_agg(distinct jsonb_build_object(
       'context', c.context,
       'contenttype', c.contenttype,
       'language', c.language,
       'base', c.base,
       'value', c.value
     )) FILTER (WHERE c.value IS NOT NULL) as content,
     json_agg(distinct jsonb_build_object(
      'id', topic.uuid,
      'name', topic.name
     )) FILTER (WHERE topic.uuid IS NOT NULL) as topics
     FROM news_article a
      INNER JOIN news_feed f ON (a.feed_id = f.id)
      INNER JOIN news_publisher p ON (p.id = f.publisher_id)
      LEFT JOIN news_article_tags at ON (at.article_id = a.id)
      LEFT JOIN view_tag_normalization t ON (t.id = at.tag_id)
      LEFT JOIN news_author w ON (w.article_id = a.id)
      LEFT JOIN news_articlecontent c ON (c.article_id = a.id)
      LEFT JOIN news_topic_tags ON (t.id = news_topic_tags.tag_id)
      LEFT JOIN news_topic topic ON (topic.id = news_topic_tags.topic_id)
      
     GROUP BY 
      a.id, a.link, a.medias, f.language, f.id, p.id, f.uuid, p.uuid
;
```
