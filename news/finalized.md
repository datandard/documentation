## Finalized news items

The database is built up in a normalized way that makes it less-scaleable to query from. To make it easier to export, we serialize the data into a table. 
Most complex data structures are saved as `JSONB` columns. This makes it a bit less easy to query, but it is doable. The database structure looks like:

```sql
CREATE TABLE news_finalized_articles (
  id integer PRIMARY KEY,
  link varchar(3000),
  media JSONB,
  published timestamp without time zone,
  langauge char(2),
  tags JSONB,
  authors JSONB,
  content JSONB,
  topics JSONB,
  feed_id integer,
  publisher_id integer
);
```

This structure makes it possible to filter based on `language`, `publisher`, `feed` and `published` without any advanced querying. 
You can query by `topic`, or `tag` by filtering through JSON.

## Triggered updates

 - Topic change
 - Tag change
 - New article

## Notes
We might have the same article many times with different article ids but same url. That is because a publisher might have the same article in multiple feeds.
We do not discard taht when we collect the data so it must be done in this aggregation. This is done by checking if we already have an article with the same
link but different id. If so, we do not add it.
