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
datandard=> select * FROM view_tag_normalization where id in (760297, 760298, 760299, 157805, 257028);
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
