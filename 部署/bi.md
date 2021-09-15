### BI



### superset

[Documentation | Superset (apache.org)](https://superset.apache.org/docs/installation/installing-superset-using-docker-compose)

```
 git clone https://github.com/apache/superset.git
 cd superset
 docker-compose -f docker-compose-non-dev.yml up
```







Mete base

https://www.metabase.com/docs/latest/operations-guide/configuring-application-database.html

[Sharing Work (metabase.com)](https://www.metabase.com/learn/getting-started/sharing-work.html)

注意数据库版本 要5.7.30 以上才能正常

```
docker run -d -p 3001:3000 --name metabase metabase/metabase


持久化数据
docker run -d -p 3000:3000 \
-v ~/metabase-data:/metabase-data \
-e "MB_DB_FILE=/metabase-data/metabase.db" \
--name metabase metabase/metabase


docker run -d -p 3000:3000 \
  -e "MB_DB_TYPE=postgres" \
  -e "MB_DB_DBNAME=metabase" \
  -e "MB_DB_PORT=5432" \
  -e "MB_DB_USER=<username>" \
  -e "MB_DB_PASS=<password>" \
  -e "MB_DB_HOST=my-database-host" \
  --name metabase metabase/metabase
  
  
  
 docker run -d --env-file .env -p 3000:3000 --name metabase metabase/metabase
  
.env
MB_DB_TYPE=mysql
MB_DB_DBNAME=metabase
MB_DB_PORT=3306
MB_DB_USER=
MB_DB_PASS=
MB_DB_HOST
  
docker run -d --restart=always -p 3000:3000 \
  -e "MB_DB_TYPE=mysql" \
  -e "MB_DB_DBNAME=metabase" \
  -e "MB_DB_PORT=33306" \
  -e "MB_DB_USER=root" \
  -e "MB_DB_PASS=123456" \
  -e "MB_DB_HOST=172.26.0.109" \
  --name metabase metabase/metabase
  
  docker run --restart=always  -p 3001:3000 \
  -e "MB_DB_TYPE=mysql" \
  -e "MB_DB_DBNAME=metabase" \
  -e "MB_DB_PORT=33307" \
  -e "MB_DB_USER=root" \
  -e "MB_DB_PASS=123456" \
  -e "MB_DB_HOST=172.26.0.109" \
  --name metabase metabase/metabase
  
  
  docker run -p 3003:3000 \
  -e "MB_DB_TYPE=mysql" \
  -e "MB_DB_DBNAME=metabase" \
  -e "MB_DB_PORT=33307" \
  -e "MB_DB_USER=root" \
  -e "MB_DB_PASS=123456" \
  -e "MB_DB_HOST=172.26.0.109" \
   metabase/metabase
  
  
export MB_DB_TYPE=mysql
export MB_DB_DBNAME=metabase
export MB_DB_PORT=33306
export MB_DB_USER=root
export MB_DB_PASS=123456
export MB_DB_HOST=localhost
export MB_JETTY_PORT=3001
```



DATABASECHANGELOG

DATABASECHANGELOGLOCK
