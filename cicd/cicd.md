git 钩子

k8s执行原理

[The Kubernetes executor | GitLab](https://docs.gitlab.com/runner/executors/kubernetes.html)



gitlab ci常量

[Predefined variables reference | GitLab](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)



gitlab token要用access token





k8s 拉去私有镜像

https://kubernetes.io/zh/docs/tasks/configure-pod-container/pull-image-private-registry/

```
注意namespace
kubectl create secret docker-registry regcred \
  --docker-server=<你的镜像仓库服务器> \
  --docker-username=<你的用户名> \
  --docker-password=<你的密码> \
  --docker-email=<你的邮箱地址>
  
 kubectl get secret regcred --output=yaml
 
 
 关联到serviceaccount
 kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'
 
 kubectl get secret regcred --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode
```



```yaml
stages:
  - release
  - deploy
release_image:
  stage: release
  tags:
    - docker
  only:
    - dev
    - test
  script:
    - docker login -u $DOCKER_REGISTRY_USERNAME -p $DOCKER_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build --pull -t $CI_REGISTRY/$CI_COMMIT_REF_NAME/$PROJECT_NAME .
    - docker push $CI_REGISTRY/$CI_COMMIT_REF_NAME/$PROJECT_NAME
deploy_to_dev_test:
  stage: deploy
  tags:
    - docker
  only:
    - dev
    - test
  script:
    - IMAGETAG=`date "+%Y%m%d"`-${CI_COMMIT_SHA:0:6}
    - echo "IMAGETAG:"$IMAGETAG
    - docker login -u $DOCKER_REGISTRY_USERNAME -p $DOCKER_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build --pull -t $CI_REGISTRY/$CI_COMMIT_REF_NAME/$PROJECT_NAME:$IMAGETAG .
    - docker push $CI_REGISTRY/$CI_COMMIT_REF_NAME/$PROJECT_NAME:$IMAGETAG
    - docker pull kubectl:$CI_COMMIT_REF_NAME
    - docker run kubectl:$CI_COMMIT_REF_NAME kubectl set image deployment/$PROJECT_NAME $PROJECT_NAME=$TEST_HUB_NAME/$CI_COMMIT_REF_NAME/$PROJECT_NAME:$IMAGETAG --namespace=$CI_COMMIT_REF_NAME
```



基本pipline

```
# This file is a template, and might need editing before it works on your project.
# To contribute improvements to CI/CD templates, please follow the Development guide at:
# https://docs.gitlab.com/ee/development/cicd/templates.html
# This specific template is located at:
# https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Bash.gitlab-ci.yml

# See https://docs.gitlab.com/ee/ci/yaml/index.html for all available options

# you can delete this line if you're not using Docker
image: busybox:latest


build1:
  image: golang:1.16.8
  stage: build
  script:
    - go version
    - ls -al


deploy1:
  image: harbor-ali.xindong.com/fanyuqi/kaniko:debug
  stage: build
  script:
    - mkdir -p /kaniko/.docker  
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(echo -n ${CI_REGISTRY_USER}:${CI_REGISTRY_PASSWORD} | base64)\"}}}" > /kaniko/.docker/config.json
    - echo $CI_REGISTRY/fanyuqi/ci-build-image:$CI_COMMIT_BRANCH
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY/fanyuqi/ci-build-image:$CI_COMMIT_BRANCH

apply1:
    image: harbor-ali.xindong.com/fanyuqi/kubectl:1.0
    stage: deploy
    script:
        - kubectl apply -f $CI_PROJECT_DIR/deployment/


```



aliyun k8s cache

https://blog.dteam.top/posts/2020-11/gitlab-ci-use-aliyun-k8s-serverless.html
