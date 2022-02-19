### Sql_mode



```
select @@GLOBAL.sql_mode
或
select @@SESSION.sql_mode



SET GLOBAL sql_mode = 'modes...';
或
SET SESSION sql_mode = 'modes...';
```