# 05 Continuous Top-N

![Twitter Badge](https://img.shields.io/badge/Flink%20Version-1.9%2B-lightgrey)

> :bulb: This example will show how to continuously calculate the "Top-N" rows based on a given attribute, using an `OVER` window and the `ROW_NUMBER()` function.

The source table (`spells_cast`) is backed by the [`faker` connector](https://flink-packages.org/packages/flink-faker), which continuously generates rows in memory based on Java Faker expressions.

The Ministry of Magic tracks every spell a wizard casts throughout Great Britain and wants to know every wizard's Top 2 all-time favorite spells. 

Flink SQL can be used to calculate continuous [aggregations](../../foundations/05_group_by/05_group_by.md), so if we know
each spell a wizard has cast, we can maintain a continuous total of how many times they have cast that spell. 

```sql
SELECT wizard, spell, COUNT(*) AS times_cast
FROM spells_cast
GROUP BY wizard, spell;
```

This result can be used in an `OVER` window to calculate a [Top-N](https://docs.ververica.com/user_guide/sql_development/queries.html#top-n).
The rows are partitioned using the `wizard` column, and are then ordered based on the count of spell casts (`times_cast DESC`). 
The built-in function `ROW_NUMBER()` assigns a unique, sequential number to each row, starting from one, according to the rows' ordering within the partition.
Finally, the results are filtered for only those rows with a `row_num <= 2` to find each wizard's top 2 favorite spells. 

Where Flink is most potent in this query is its ability to issue retractions.
As wizards cast more spells, their top 2 will change. 
When this occurs, Flink will issue a retraction, modifying its output, so the result is always correct and up to date. 


```sql
CREATE TABLE spells_cast (
    wizard STRING,
    spell  STRING
) WITH (
  'connector' = 'faker',
  'fields.wizard.expression' = '#{harry_potter.characters}',
  'fields.spell.expression' = '#{harry_potter.spells}'
);

SELECT wizard, spell, times_cast
FROM (
    SELECT *,
    ROW_NUMBER() OVER (PARTITION BY wizard ORDER BY times_cast DESC) AS row_num
    FROM (SELECT wizard, spell, COUNT(*) AS times_cast FROM spells_cast GROUP BY wizard, spell)
)
WHERE row_num <= 2;  
```

![05_top_n](https://user-images.githubusercontent.com/23521087/105503736-3e653700-5cc7-11eb-9ddf-9a89d93841bc.png)

## tips
### TopN algorithm [Optimize the TopN algorithm](https://www.alibabacloud.com/help/en/flink/recommended-flink-sql-practices#section-b0v-rga-gqc)
If the input streams of TopN are static streams (such as source), TopN supports only one algorithm: AppendRank. If the input streams of TopN are dynamic streams (such as streams that are processed by using the AGG or JOIN function), TopN supports the following three algorithms in descending order of performance: UpdateFastRank, UnaryUpdateRank, and RetractRank. The name of the algorithm used is contained in the node name in the topology.

- UpdateFastRank is the optimal algorithm.
The following two conditions must be met if you want to use this algorithm:
1. The input streams must contain the primary key information, such as ORDER BY AVG.
2. The values of the fields or functions in the ORDER BY clause are updated monotonically in the opposite order of sorting. For example, you can define the ORDER BY clause as ORDER BY COUNT, ORDER BY COUNT_DISTINCT, or ORDER BY SUM (positive) DESC. 
> ps:topN语句中 order by 后面的字段一定要用AVG函数生成的字段