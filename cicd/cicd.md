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

