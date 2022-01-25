### 官方文档

https://grafana.com/docs/grafana/latest/whatsnew/



```bash
docker run -d -p 3000:3000 grafana/grafana


# 插件
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -e "GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource" \
  grafana/grafana
  
# 自定义地址
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -e "GF_INSTALL_PLUGINS=http://plugin-domain.com/my-custom-plugin.zip;custom-plugin,grafana-clock-panel" \
  grafana/grafana
  
```

