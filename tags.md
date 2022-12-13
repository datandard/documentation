## Tag normalizations
This defines a view that only contains normalized tags. This takes several things into account:

 - Replacement of tags when tags are the same but spelled differently.
 - Tags that are marked as uncategorised.
 - Tags that are marked as invalid.
 - 


```sql
CREATE OR REPLACE VIEW view_tag_normalization AS 
  SELECT 
    t1.id as id, COALESCE(t2.term, t1.term) as term 
  FROM news_tag t1 
    LEFT JOIN news_tag t2 ON t1.replaced_by_id = t2.id
  WHERE
    (t1.is_uncategorised = 'f' AND t2.is_uncategorised = 'f')
    AND (t1.valid = 't' AND t2.valid = 't')
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
