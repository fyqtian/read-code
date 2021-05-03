### Gitlab

https://docs.gitlab.com/ee/README.html



sudo docker pull gitlab/gitlab-runner:latest



```
docker run -d --name gitlab-runner --restart always \
  -v /opt/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
  
配置在git cicd setting
docker exec -it gitlab-runner gitlab-ci-multi-runner register


```


