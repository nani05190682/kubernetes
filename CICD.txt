stages:
  - build
  - test
  - deploy_dev
  - deploy_test
  - deploy_prod

variables:
  APP_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  KUBE_NAMESPACE_DEV: dev-namespace
  KUBE_NAMESPACE_TEST: test-namespace
  KUBE_NAMESPACE_PROD: prod-namespace

before_script:
  - echo "Logging into GitLab Container Registry..."
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - echo "Building Docker image..."
    - docker build -t $APP_IMAGE .
    - docker push $APP_IMAGE
  only:
    - main

test:
  stage: test
  script:
    - echo "Running unit tests..."
    - ./run-tests.sh
  only:
    - main

deploy_dev:
  stage: deploy_dev
  script:
    - echo "Deploying to Development Environment..."
    - kubectl set image deployment/my-app my-app=$APP_IMAGE -n $KUBE_NAMESPACE_DEV
    - kubectl rollout status deployment/my-app -n $KUBE_NAMESPACE_DEV
  only:
    - main

deploy_test:
  stage: deploy_test
  script:
    - echo "Deploying to Test Environment..."
    - kubectl set image deployment/my-app my-app=$APP_IMAGE -n $KUBE_NAMESPACE_TEST
    - kubectl rollout status deployment/my-app -n $KUBE_NAMESPACE_TEST
  only:
    - main

deploy_prod:
  stage: deploy_prod
  script:
    - echo "Deploying to Production Environment..."
    - kubectl set image deployment/my-app my-app=$APP_IMAGE -n $KUBE_NAMESPACE_PROD
    - kubectl rollout status deployment/my-app -n $KUBE_NAMESPACE_PROD
  only:
    - main
  when: manual
  environment:
    name: production
    url: https://prod.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
      allow_failure: false
