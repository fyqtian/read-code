# mongodb







### 创建用户

```
注意切换db
db.createUser(
     {
       user:"yapi",
       pwd:"5OozaxswAQFgZDeI",
       roles:[{role:"readWrite",db:"yapi"}]
     }
  )


```





#### 链接

```
mongo "mongodb://yapi:5OozaxswAQFgZDeI@dds-uf6373b66cae9b041.mongodb.rds.aliyuncs.com:3717,dds-uf6373b66cae9b042.mongodb.rds.aliyuncs.com:3717/yapi?replicaSet=mgset-48394868"
```

