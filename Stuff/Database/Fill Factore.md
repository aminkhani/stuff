## Optimizing PostgreSQL Performance: How the Fill Factor Setting improve I/O performance

The `fillfactor` is a **storage parameter** in PostgreSQL that defines the **percentage of space** on each page of a table or index that can be filled with data during **insert** operations. The remaining space (the percentage left after the fill factor) is **reserved for future updates**, reducing the need for page splits or row movement, which can be expensive in terms of performance.

---
### How Fill Factor Works?

**Default Value:** The default `fillfactor` for tables is **100**, meaning that PostgreSQL will attempt to fill each page completely during initial inserts. For indexes, the default is often set to **90**, meaning 90% of the page will be filled.

- Setting the Fill Factor: You can set the `fillfactor` for a table or an index when creating or altering them. For example:

```sql
CREATE TABLE my_table (  
id serial PRIMARY KEY,  
data text  
) WITH (fillfactor = 70);
```
This sets a fill factor of 70 for the `my_table` table, meaning only 70% of each page will be filled during initial inserts.

---
## **Impact on Performance:**

- **Low Fill Factor:** If the fill factor is set low (e.g., 70%), there is more room for future updates on each page, reducing the likelihood of needing to move rows or split pages. This is particularly useful for tables with frequent **updates**.  
- **High Fill Factor:** A higher fill factor (e.g., 100%) means less free space on each page. This might be efficient for tables that are primarily read-only after being populated, as it minimizes disk space usage. However, it can lead to more page splits or row movement if the table is frequently updated, potentially degrading performance.
---
