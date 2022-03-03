

https://hub.docker.com/_/postgres

https://github.com/postgres-cn/pgdoc-cn

http://www.postgres.cn/docs/13/

```shell
docker run --name some-postgres -e POSTGRES_PASSWORD=123456  -p 5432:5432 -d postgres
docker run --name postgresql -p 5432:5432 -e POSTGRES_PASSWORD=123456 -e PGPASSWORD=123456 -d postgres

命令行
psql -U postgres -W 
psql -h localhost -p 5432 -U postgres {dbname}

docker run -d \
    --name some-postgres \
    -e POSTGRES_PASSWORD=mysecretpassword \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v /custom/mount:/var/lib/postgresql/data \
    postgres
    
    
docker pull dpage/pgadmin4:4.17
docker run --name pgadmin -p 5080:80 \
    -e 'PGADMIN_DEFAULT_EMAIL=pekkle@abc.com' \
    -e 'PGADMIN_DEFAULT_PASSWORD=123456' \
    -e 'PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True' \
    -e 'PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!"' \
    -e 'PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10' \
    -d dpage/pgadmin4
    
这是postgresql数据库的trust认证设计，即任意os用户无需密码就可以获得postgresql数据库管理员权限，不检查os用户名，用户组。这被很多程序员认为是PG数据库的一个安全漏洞。
可以通过配置pg_hba.conf和pg_ident.conf文件禁止os用户以trust认证方式登录。
```





### sql

```sql
CREATE DATABASE runoobdb
    WITH 
    OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TABLESPACE = pg_default
    CONNECTION LIMIT = -1;



createdb 命令位于 PostgreSQL安装目录/bin 下，执行创建数据库的命令：

createdb [option...] [dbname [description]]

DROP DATABASE [ IF EXISTS ] name;

dropdb -h localhost -p 5432 -U postgres runoobdb

DROP TABLE table_name,table2;

\l   								查看数据库

\c + 数据库名        进入数据库

\d 	{tablename}			查看表结构

 \d+ 

CREATE ROLE postgres WITH
  LOGIN
  SUPERUSER
  INHERIT
  CREATEDB
  CREATEROLE
  REPLICATION
  ENCRYPTED PASSWORD 'SCRAM-SHA-256$4096:ew6IocbH99auZg44ksnK7Q==$VaLA837sytpOqXtMXosmML+yioCTRkyRoIuKZxwvFPg=:OiMmgbi+CVB4TNQ3Hpjl8aL2Dw0HSf0RHPvtSrplU7E=';


CREATE TABLE table_name(
   column1 datatype,
   column2 datatype,
   column3 datatype,
   .....
   columnN datatype,
   PRIMARY KEY( 一个或多个列 )
);

CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);

CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL,
   JOIN_DATE      DATE
);


INSERT INTO TABLE_NAME VALUES (value1,value2,value3,...valueN);

INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY,JOIN_DATE) VALUES (1, 'Paul', 32, 'California', 20000.00,'2001-07-13');

INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,JOIN_DATE) VALUES (2, 'Allen', 25, 'Texas', '2007-12-13');


INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY,JOIN_DATE) VALUES (4, 'Mark', 25, 'Rich-Mond ', 65000.00, '2007-12-13' ), (5, 'David', 27, 'Texas', 85000.00, '2007-12-13');


SELECT * FROM public.company ORDER BY id ASC 

UPDATE table_name SET column1 = value1, column2 = value2...., columnN = valueN WHERE [condition];

DELETE FROM table_name WHERE [condition];


SELECT FROM table_name WHERE column LIKE '%XXXX%';
或者
SELECT FROM table_name WHERE column LIKE 'XXXX_';
```





### PostgreSQL 模式（SCHEMA）

```sql
PostgreSQL 模式（SCHEMA）可以看着是一个表的集合。

一个模式可以包含视图、索引、数据类型、函数和操作符等。

相同的对象名称可以被用于不同的模式中而不会出现冲突，例如 schema1 和 myschema 都可以包含名为 mytable 的表。

使用模式的优势：

允许多个用户使用一个数据库并且不会互相干扰。

将数据库对象组织成逻辑组以便更容易管理。

第三方应用的对象可以放在独立的模式中，这样它们就不会与其他对象的名称发生冲突。

create schema myschema;

create table myschema.company(
   ID   INT              NOT NULL,
   NAME VARCHAR (20)     NOT NULL,
   AGE  INT              NOT NULL,
   ADDRESS  CHAR (25),
   SALARY   DECIMAL (18, 2),
   PRIMARY KEY (ID)
);

删除一个为空的模式（其中的所有对象已经被删除）：

DROP SCHEMA myschema;
删除一个模式以及其中包含的所有对象：

DROP SCHEMA myschema CASCADE;
```





### 约束

```sql
UNIQUE
CREATE TABLE COMPANY3(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL UNIQUE,
   ADDRESS        CHAR(50),
   SALARY         REAL    DEFAULT 50000.00
);

CHECK
CREATE TABLE COMPANY5(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL    CHECK(SALARY > 0)
);

CREATE TABLE COMPANY7(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT,
   AGE            INT  ,
   ADDRESS        CHAR(50),
   SALARY         REAL,
   EXCLUDE USING gist
   (NAME WITH =,  -- 如果满足 NAME 相同，AGE 不相同则不允许插入，否则允许插入
   AGE WITH <>)   -- 其比较的结果是如果整个表边式返回 true，则不允许插入，否则允许
);

ALTER TABLE table_name DROP CONSTRAINT some_name;
```



### 权限

```sql
CREATE USER runoob WITH PASSWORD 'password';
GRANT ALL ON COMPANY TO runoob;
REVOKE ALL ON COMPANY FROM runoob;
DROP USER runoob;

CREATE DATABASE dbname OWNER rolename;

// 看在那个database下面

select * from information_schema.table_privileges where grantee='fyq';
1、查看某用户的表权限
select * from information_schema.table_privileges where grantee='user_name';
2、查看usage权限表
select * from information_schema.usage_privileges where grantee='user_name';
3、查看存储过程函数相关权限表
select * from information_schema.routine_privileges where grantee='user_name';

```





```sql
SELECT version();
SELECT current_date;

继承
CREATE TABLE cities (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE TABLE capitals (
  state      char(2) UNIQUE NOT NULL
) INHERITS (cities);


$ psql -s mydb
psql's -s option puts you in single step mode which pauses before sending each statement to the server. The commands used in this section are in the file basics.sql.'
mydb=> \i basics.sql


COPY weather FROM '/home/user/weather.txt';


distinct 或默认排序 老版本pg会 现在不确定

 In some database systems, including older versions of PostgreSQL, the implementation of DISTINCT automatically orders the rows and so ORDER BY is unnecessary. But this is not required by the SQL standard, and current PostgreSQL does not guarantee that DISTINCT causes the rows to be ordered.
 
 
 
 CREATE TABLE products (
    product_no integer DEFAULT nextval('products_product_no_seq'),
);

CREATE TABLE products (
    product_no SERIAL
);


A generated column cannot be written to directly. In INSERT or UPDATE commands, a value cannot be specified for a generated column, but the keyword DEFAULT may be specified.

CREATE TABLE people (
    ...,
    height_cm numeric,
    height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```



### check

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0)
);

CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CONSTRAINT positive_price CHECK (price > 0)
);

CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric CHECK (discounted_price > 0),
    CHECK (price > discounted_price)
);

CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0 AND price > discounted_price)
);

CREATE TABLE products (
    product_no integer NOT NULL,
    name text NOT NULL,
    price numeric
);

CREATE TABLE products (
    product_no integer UNIQUE,
    name text,
    price numeric
);

CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    UNIQUE (product_no)
);

CREATE TABLE example (
    a integer,
    b integer,
    c integer,
    UNIQUE (a, c)
);
```



### with

```sql
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total_sales
    FROM orders
    GROUP BY region
), top_regions AS (
    SELECT region
    FROM regional_sales
    WHERE total_sales > (SELECT SUM(total_sales)/10 FROM regional_sales)
)
SELECT region,
       product,
       SUM(quantity) AS product_units,
       SUM(amount) AS product_sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;

WITH RECURSIVE t(n) AS (
    VALUES (1)
  UNION ALL
    SELECT n+1 FROM t WHERE n < 100
)
SELECT sum(n) FROM t;
```









### ddl

https://www.postgresql.org/docs/14/ddl-alter.html



### dml

https://www.postgresql.org/docs/14/dml.html

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);

INSERT INTO products VALUES (1, 'Cheese', 9.99);

INSERT INTO products DEFAULT VALUES;

INSERT INTO products (product_no, name, price) VALUES
    (1, 'Cheese', 9.99),
    (2, 'Bread', 1.99),
    (3, 'Milk', 2.99);
    
INSERT INTO products (product_no, name, price)
  SELECT product_no, name, price FROM new_products
    WHERE release_date = 'today';
    
UPDATE products SET price = 10 WHERE price = 5;

UPDATE products SET price = price * 1.10;

UPDATE mytable SET a = 5, b = 3, c = 1 WHERE a > 0;

DELETE FROM products WHERE price = 10;

## careful
DELETE FROM products;

INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool') RETURNING id;
```



### query

```sql
[WITH with_queries] SELECT select_list FROM table_expression [sort_specification]

SELECT a, b + c FROM table1;

FROM (SELECT * FROM table1) AS alias_name

SELECT ... FROM fdt WHERE c1 > 5

SELECT ... FROM fdt WHERE c1 IN (1, 2, 3)

SELECT ... FROM fdt WHERE c1 IN (SELECT c1 FROM t2)

SELECT ... FROM fdt WHERE c1 IN (SELECT c3 FROM t2 WHERE c2 = fdt.c1 + 10)

SELECT ... FROM fdt WHERE c1 BETWEEN (SELECT c3 FROM t2 WHERE c2 = fdt.c1 + 10) AND 100

SELECT ... FROM fdt WHERE EXISTS (SELECT c1 FROM t2 WHERE c2 > fdt.c1)



GROUPING SETS, CUBE, and ROLLUP


(DISTINCT ON processing occurs after ORDER BY sorting
 
 
union会做distinct操作
UNION effectively appends the result of query2 to the result of query1 (although there is no guarantee that this is the order in which the rows are actually returned). Furthermore, it eliminates duplicate rows from its result, in the same way as DISTINCT, unless UNION ALL is used.
```



## `VALUES` Lists

```sql
VALUES (1, 'one'), (2, 'two'), (3, 'three');

SELECT 1 AS column1, 'one' AS column2
UNION ALL
SELECT 2, 'two'
UNION ALL
SELECT 3, 'three';

SELECT * FROM (VALUES (1, 'one'), (2, 'two'), (3, 'three')) AS t (num,letter);
```







### 数据分区

https://www.postgresql.org/docs/14/ddl-partitioning.html



```
强制删除
DROP TABLE products CASCADE;
```





```
\l
\d+
\di               打印索引
\df
\timing on
\dn     					列出scheme
\db 							列出表空间
\du  |  \dg				列出用户
\dp t | \z        显示表的权限分配


\dt              查看表
\d test1


\pset  border 0
\pset  border 1
\pset  border 2


\x     数据拆分单行

\i     执行外部文件     =   psql -x -f|{-s} filename

\c db1 lottu2

psql -E     打印详细的命令
```





### 权限

https://developer.aliyun.com/article/41210

https://www.cnblogs.com/lottu/p/12916046.html

https://www.jb51.net/article/205417.htm

https://www.cnblogs.com/zhoujinyi/p/10939715.html

https://www.modb.pro/db/103980

https://www.modb.pro/db/102674



### select * from pg_user;

| `usename`      | `name`        | 用户名                                                       |
| -------------- | ------------- | ------------------------------------------------------------ |
| `usesysid`     | `oid`         | 用户的ID                                                     |
| `usecreatedb`  | `bool`        | 用户是否能创建数据库                                         |
| `usesuper`     | `bool`        | 用户是否为超级用户                                           |
| `userepl`      | `bool`        | 用户能否开启流复制以及将系统转入/转出备份模式。              |
| `usebypassrls` | `bool`        | 用户能否绕过所有的行级安全性策略，详见 [第 5.8 节](https://www.modb.pro/db/ddl-rowsecurity.html)。 |
| `passwd`       | `text`        | 不是口令（总是显示为`********`）                             |
| `valuntil`     | `timestamptz` | 口令过期时间（只用于口令认证）                               |
| `useconfig`    | `text[]`      | 运行时配置变量的会话默认值                                   |



### select * from pg_authid ;

| `oid`            | `oid`         | 行标识符                                                     |
| ---------------- | ------------- | ------------------------------------------------------------ |
| `rolname`        | `name`        | 角色名                                                       |
| `rolsuper`       | `bool`        | 角色是否拥有超级用户权限                                     |
| `rolinherit`     | `bool`        | 如果本角色是另一个角色的成员，本角色是否自动另一个角色的权限 |
| `rolcreaterole`  | `bool`        | 角色是否能创建更多角色                                       |
| `rolcreatedb`    | `bool`        | 角色是否能创建数据库                                         |
| `rolcanlogin`    | `bool`        | 角色是否能登录。即该角色是否能够作为初始会话授权标识符       |
| `rolreplication` | `bool`        | 角色是一个复制角色。复制角色可以启动复制连接并且创建和删除复制槽。 |
| `rolbypassrls`   | `bool`        | 角色是否可以绕过所有的行级安全性策略，详见 [第 5.8 节](https://www.modb.pro/db/ddl-rowsecurity.html)。 |
| `rolconnlimit`   | `int4`        | 对于可以登录的角色，本列设置该角色可以同时发起最大连接数。-1表示无限制。 |
| `rolpassword`    | `text`        | 口令（可能被加密过），如果没有口令则为空。格式取决于使用的加密方法的形式。 |
| `rolvaliduntil`  | `timestamptz` | 口令过期时间（只用于口令鉴定），如果永不过期则为空           |







```sql
在数据库中所有的权限都和角色（用户）挂钩，public是一个特殊角色，代表所有人。

超级用户是有允许任意操作对象的，普通用户只能操作自己创建的对象。

另外有一些对象是有赋予给public角色默认权限的，所以建好之后，所以人都有这些默认权限。

SELECT rolname FROM pg_roles;
\du
\duS+  List of roles
\l+   查看库
\dp    Access privileges
\d+     List of relations
\dn     List of schemas

select * from pg_tables where schemaname='f1';


select current_user;

select relname,relacl from pg_class where relkind='r';

revoke create on schema public from PUBLIC;

grant select on  all tables in schema public to username;

alter default privileges in scheme  public grant select on tables to username;

revoke CONNECT ON DATABASE db1 from public;

grant CONNECT ON DATABASE db1 to lottu2;

grant USAGE ON SCHEMA lottu1 to lottu2;

grant select on TABLE tbl_lottu_01 to lottu2;

grant select on ALL TABLES IN SCHEMA lottu1 to lottu2;

rolename=xxxx -- privileges granted to a role
        =xxxx -- privileges granted to PUBLIC

            r -- SELECT ("read")
            w -- UPDATE ("write")
            a -- INSERT ("append")
            d -- DELETE
            D -- TRUNCATE
            x -- REFERENCES
            t -- TRIGGER
            X -- EXECUTE
            U -- USAGE
            C -- CREATE
            c -- CONNECT
            T -- TEMPORARY
      arwdDxt -- ALL PRIVILEGES (for tables, varies for other objects)
            * -- grant option for preceding privilege
            
create schema histroy;
revoke all on schema from public;
grant all on schema to somerole;
```





### index

https://docs.postgresql.tw/the-sql-language/index/index-types

https://developer.aliyun.com/article/111793





### 全文索引

```sql
SELECT 'a fat cat sat on a mat and ate a fat rat'::tsvector @@ 'cat & rat'::tsquery;
 ?column?
```





### explain

```
EXPLAIN SELECT * FROM tenk1;
EXPLAIN analyze SELECT * FROM tenk1;
explain (analyze true,buffers true) select * from xxx
explain (format json) select * from products
```

 Seq Scan on tenk1  (cost=0.00..458.00 rows=10000 width=244)



被包含在圆括号中的数字是（从左至右）

- 估计的启动开销。在输出阶段可以开始之前消耗的时间，例如在一个排序结点里执行排序的时间。

- 估计的总开销。这个估计值基于的假设是计划结点会被运行到完成，即所有可用的行都被检索。不过实际上一个结点的父结点可能很快停止读所有可用的行（见下面的LIMIT例子）。

- 这个计划结点输出行数的估计值。同样，也假定该结点能运行到完成。

- 预计这个计划结点输出的行平均宽度（以字节计算）。



```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'tenk1';
```



### 服务器参数

http://www.postgres.cn/docs/13/config-setting.html



配置文件

/var/lib/postgresql/data

pg_ctl reload 		重新加载配置文件

 `postgresql.auto.conf`中的设置会覆盖`postgresql.conf`中的设置。



#### show all 查看所有参数



编辑`postgresql.conf`文件， 它通常被保存在数据目录中（当数据库集簇目录被初始化时，一个默认的拷贝将会被安装在那里）。一个该文件的例子看起来是：





### 磁盘空间

 https://www.cnblogs.com/itcomputer/articles/5149222.html  生成测试数据

```sql
SELECT pg_relation_filepath(oid), relpages FROM pg_class WHERE relname = 'tablename';

insert INTO ff   SELECT * FROM generate_series(1,500000000);

// 找索引
SELECT c2.relname, c2.relpages
FROM pg_class c, pg_class c2, pg_index i
WHERE c.relname = '{table}' AND
      c.oid = i.indrelid AND
      c2.oid = i.indexrelid
ORDER BY c2.relname;

//找表
SELECT relname, relpages
FROM pg_class
ORDER BY relpages DESC;
```



### 膨胀



http://mysql.taobao.org/monthly/2015/12/07/

https://developer.aliyun.com/article/463916

https://www.modb.pro/db/25830

pg_repack

```sql
 select * from pg_stat_activity where state<>'idle' and pg_backend_pid() != pid and (backend_xid is not null or backend_xmin is not null ) and extract(epoch from (now() - xact_start))  > <时间阈值，单位秒> ;
 
select * from pg_stat_activity;
```

