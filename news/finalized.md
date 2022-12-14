## Finalized news items

The database is built up in a normalized way that makes it less-scaleable to query from. To make it easier to export, we serialize the data into a table. 
Most complex data structures are saved as `JSONB` columns. This makes it a bit less easy to query, but it is doable. The database structure looks like:

```sql
CREATE TABLE news_finalized_articles (
  id integer PRIMARY KEY,
  uuid uuid,
  link varchar(3000),
  media JSONB,
  published timestamp without time zone,
  feed_id bigint,
  feed_uuid uuid,
  feed_language varchar(3000),
  publisher_id bigint,
  publisher_uuid uuid,
  language varchar(3000),
  tags JSONB,
  authors JSONB,
  content JSONB,
  topics JSONB
);
```

This structure makes it possible to filter based on `language`, `publisher`, `feed` and `published` without any advanced querying. 
You can query by `topic`, or `tag` by filtering through JSON.

For updating purpose we create an index on link:

```sql
create index news_finalized_articles_link ON news_finalized_articles(link);
```

## Triggered updates
When a new article is inserted into the system the id is added to the `NEWS_ARTICLE_FINALIZER` queue, where it is picked up by a worker thread that will insert into the finalized form (see below section). An article will also be inserted or updated when a topic is changed. When a tag is added this is always happening before a new article is inserted so there is no need to update it. 


## Building the final object
The query to build the finalized object is stored as a view. This make the query to replace them in the script simple:

```sql
INSERT INTO news_finalized_articles SELECT * FROM view_articles LIMIT 1
```

See [Finalized article view](news/finalized_view.md) documentation for how this works:

## How to query the table

### Filtering by topics
As an example, see the following subset:

```sql
datandard=> select id, topics from news_finalized_articles where id in (7785318, 7783263, 7783556, 7783795);
   id    |                                                                 topics
---------+-----------------------------------------------------------------------------------------------------------------------------------------
 7783263 | [{"id": "ddded8f7-1a72-4e43-b040-86a01e82d2c6", "name": "Finance"}]
 7783556 | [{"id": "7bad8662-a07b-45c5-bea5-1aa6050c0dfb", "name": "Politics"}]
 7783795 |
 7785318 | [{"id": "7bad8662-a07b-45c5-bea5-1aa6050c0dfb", "name": "Politics"}, {"id": "ddded8f7-1a72-4e43-b040-86a01e82d2c6", "name": "Finance"}]
(4 rows)

```


## Notes
We might have the same article many times with different article ids but same url. That is because a publisher might have the same article in multiple feeds.
We do not discard taht when we collect the data so it must be done in this aggregation. This is done by checking if we already have an article with the same
link but different id. If so, we do not add it.
