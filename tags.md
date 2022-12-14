## Tag normalizations
This defines a view that only contains normalized tags. This takes several things into account:

 - Replacement of tags when tags are the same but spelled differently.
 - Tags that are marked as uncategorised.
 - Tags that are marked as invalid.


```sql
CREATE OR REPLACE VIEW view_tag_normalization AS 
  SELECT 
    t1.id as id, 
    t1.uuid as uuid,
    COALESCE(t2.term, t1.term) as term, 
    COALESCE(t2.language, t1.language) as language 
  FROM news_tag t1 
    LEFT JOIN news_tag t2 ON t1.replaced_by_id = t2.id AND t2.is_uncategorised = 'f' AND t2.valid = 't'
  WHERE
    t1.is_uncategorised = 'f' AND t1.valid = 't';
```

This creates a table with duplicate tags but keeping the original ids:

```sql
datandard=> SELECT * FROM view_tag_normalization LIMIT 5;
datandard=> select * from view_tag_normalization limit 5 offset 300;
   id    |       term       | language
---------+------------------+----------
   53481 | Boycotts         | en
 1073986 | Diaries          | af
  764913 | Atlantic Council | en
 1001031 | Die Zeit         | de
  224513 | Sky News         | pl
(5 rows)

The conditions in the join makes sure only valid tags can overtake another tag. 
