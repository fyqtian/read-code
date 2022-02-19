### Gitlab

https://docs.gitlab.com/ee/README.html



sudo docker pull gitlab/gitlab-runner:latest



shell

```
curl -o gitlab-runner https://s3.amazonaws.com/gitlab-runner-downloads/master/binaries/gitlab-runner-linux-amd64
chmod +x gitlab-runner
```

[Run GitLab Runner in a container (Docker in docker)_Jack的博客-CSDN博客](https://blog.csdn.net/windy135/article/details/103885052) docker in docker

```
docker run -d --name gitlab-runner --restart always \
  -v /opt/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
  
配置在git cicd setting
docker exec -it gitlab-runner gitlab-ci-multi-runner register

sudo gitlab-runner register -n \
  --url https://gitlab.com/ \
  --registration-token REGISTRATION_TOKEN \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "docker:19.03.12" \
  --docker-privileged \

  
  
  sudo gitlab-runner register -n \
  --url  \
  --registration-token  \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "docker:19.03.12" \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock
  
  
 [[runners]]
  url = "https://gitlab.com/"
  token = TOKEN
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "docker:19.03.12"
    privileged = true
    disable_cache = false
    volumes = ["/cache"]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]

```





Gitlab-runner

```
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "fyq-runner"
  url = "https://git.xindong.com/"
  token = "9EsF487WCxv16Ve6z7p1"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "golang:1.16.3"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    pull_policy = ["if-not-present"]
    shm_size = 0

[[runners]]
  name = "other"
  url = "https://git.xindong.com/"
  token = "qMVcwv1eyDqBozE1k4x5"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker:19.03.12"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    pull_policy = ["if-not-present"]
    shm_size = 0
```



基本流水线

```
stages:
  - build
  - test
  - deploy

image: alpine

build_a:
  stage: build
  script:
    - echo "This job builds something."

build_b:
  stage: build
  script:
    - echo "This job builds something else."

test_a:
  stage: test
  script:
    - echo "This job tests something. It will only run when all jobs in the"
    - echo "build stage are complete."

test_b:
  stage: test
  script:
    - echo "This job tests something else. It will only run when all jobs in the"
    - echo "build stage are complete too. It will start at about the same time as test_a."

deploy_a:
  stage: deploy
  script:
    - echo "This job deploys something. It will only run when all jobs in the"
    - echo "test stage complete."

deploy_b:
  stage: deploy
  script:
    - echo "This job deploys something else. It will only run when all jobs in the"
    - echo "test stage complete. It will start at about the same time as deploy_a."
```

依赖关系

```
stages:
  - build
  - test
  - deploy

image: alpine

build_a:
  stage: build
  script:
    - echo "This job builds something quickly."

build_b:
  stage: build
  script:
    - echo "This job builds something else slowly."

test_a:
  stage: test
  needs: [build_a]
  script:
    - echo "This test job will start as soon as build_a finishes."
    - echo "It will not wait for build_b, or other jobs in the build stage, to finish."

test_b:
  stage: test
  needs: [build_b]
  script:
    - echo "This test job will start as soon as build_b finishes."
    - echo "It will not wait for other jobs in the build stage to finish."

deploy_a:
  stage: deploy
  needs: [test_a]
  script:
    - echo "Since build_a and test_a run quickly, this deploy job can run much earlier."
    - echo "It does not need to wait for build_b or test_b."

deploy_b:
  stage: deploy
  needs: [test_b]
  script:
    - echo "Since build_b and test_b run slowly, this deploy job will run much later."
```



```
job1:
  script: "execute-script-for-job1"

job2:
  script: "execute-script-for-job2"
```





```

image: alpine:latest
 
pages:
  stage: deploy
  script:
  - echo 'Nothing to do...'
  artifacts:
    paths:
    - public
  only:
  - master
  
deploytest:
  stage: deploy
  script:
  - echo 'deploy test'
  artifacts:
    paths:
    - public
  only:
  - master
 
  
deployuat:
  stage: deploy
  script:
  - echo 'deploy uat'
  artifacts:
    paths:
    - public
  only:
  - master
 
 
myjobs:
 stage: build
 script:
 - echo 'execute myjobs build'
 
testjob2:
 stage: test
 script:
 - echo 'execute mytest jon test2'
 
testjob:
 stage: test
 script:
 - echo 'execute mytest jon test'
 
firstBuild:
 stage: build
 script:
 - echo 'firstBuild'
 
firstTest:
 stage: test
 script:
 - echo 'first test'

```



### gitlab架构

https://blog.csdn.net/liumiaocn/article/details/107926262

https://chegva.com/3229.html



https://docs.gitlab.com/14.7/omnibus/settings/





### nfs

https://www.huweihuang.com/linux-notes/tools/nfs-usage.html
