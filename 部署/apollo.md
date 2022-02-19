

### Apollo

**架构设计**

https://mp.weixin.qq.com/s/-hUaQPzfsl9Lm3IqQW3VDQ

[分布式部署指南 · ctripcorp/apollo Wiki (github.com)](https://github.com/ctripcorp/apollo/wiki/分布式部署指南)

1. **ConfigService** (port 8080)

2. - 提供配置获取接口
   - 提供配置推送接口
   - 服务于Apollo客户端

3. **AdminService** (port 8090)

4. - 提供配置管理接口
   - 提供配置修改发布接口
   - 服务于管理界面Portal

5. **Client**

6. - 为应用获取配置，支持实时更新
   - 通过MetaServer获取ConfigService的服务列表
   - 使用客户端软负载SLB方式调用ConfigService

7. **Portal** (port 8070)

8. - 配置管理界面
   - 通过MetaServer获取AdminService的服务列表
   - 使用客户端软负载SLB方式调用AdminService



流程

config server注册





1. **Eureka**

2. - 用于服务发现和注册
   - Config/AdminService注册实例并定期报心跳
   - 和ConfigService住在一起部署

3. **MetaServer**

4. - Portal通过域名访问MetaServer获取AdminService的地址列表
   - Client通过域名访问MetaServer获取ConfigService的地址列表
   - 相当于一个Eureka Proxy
   - 逻辑角色，和ConfigService住在一起部署

5. **NginxLB**

6. - 和域名系统配合，协助Portal访问MetaServer获取AdminService地址列表
   - 和域名系统配合，协助Client访问MetaServer获取ConfigService地址列表
   - 和域名系统配合，协助用户访问Portal进行配置管理









### API

https://www.apolloconfig.com/#/zh/usage/other-language-client-user-guide

http

获取配置

 {config_server_url}/configfiles/json/{appId}/{clusterName}/{namespaceName}?ip={clientIp}



获取节点信息

curl "http://172.20.20.27:8080/services/config?appId=booster"