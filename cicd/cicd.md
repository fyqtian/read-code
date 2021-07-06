git 钩子





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

