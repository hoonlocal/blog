## Issue
회사에 **서비스중인 플랫폼**이 많다보니 그에 따른 **어드민 사이트도 다양하게 운영**중이다.
그 중 **호텔 시스템**을 관리하는 **서비스**가 <code>Nuxt.js(vue)</code>로 되어 있는데, 선임 개발자분께서 <code>Next.js(react)</code>로 **migration** 하는 도중에 퇴사하게 되셨다.
작업 초기에 나가게 되어 **배포 프로세스가 구축되지 않은 상황**이고, 기획, 디자인, 개발 등 추가 요청이 들어오면 **일일이 수동 배포**를 하던가 **자리로 불러서 확인**시켜 드려야 했다.
서로 여간 **불편한 일**이 아니라서 이번에 **배포 자동화**를 **구축**해보려고 한다.

<br>

## Problem
+ 회사 서비스들이 주로 **AWS ECS** 또는 **EB**로 구성
+ 현재 **쉘 스크립트**를 사용해 **ECR**에 **Docker** 이미지 배포
+ 이후 콘솔에서 **ECS** **클러스터 업데이트** 및 **태스크 재실행**
+ **ECS**와 **EB** 중 어느 서비스로 배포할지 선택해야 함
+ **Github Actions** .yml 파일 작성법이 익숙하지 않음

> _AS-IS_

```bash
# Dockerfile

FROM node:16.13.2-alpine

ENV HOST 0.0.0.0

WORKDIR /app

COPY package.json ./
COPY yarn.lock ./

RUN echo "BASE_URL=process.env.BASE_URL" >> .env
RUN echo "API_KEY=process.env.API_KEY" >> .env

RUN yarn

COPY . .

RUN yarn build

EXPOSE 3000

CMD [ "yarn", "start" ]
```

```bash
# deploy.sh
# sh ./deploy.sh
# ECR 배포 완료 후 ECS 클러스터 업데이트 및 태스크 재실행

aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 292885173959.dkr.ecr.ap-northeast-2.amazonaws.com

branch=$(git symbolic-ref --short HEAD)
docker build -t staypia-admin-web/${branch} .
docker tag staypia-admin-web/${branch}:latest 292885173959.dkr.ecr.ap-northeast-2.amazonaws.com/staypia-admin-web/${branch}:latest
docker push 292885173959.dkr.ecr.ap-northeast-2.amazonaws.com/staypia-admin-web/${branch}:latest
docker rmi --force $(docker images -q 292885173959.dkr.ecr.ap-northeast-2.amazonaws.com/staypia-admin-web/${branch}:latest)
```

<br>

## Solution
**최종적으로 배포 서비스는 ECS와 EB 중에 ECS를 사용하기로 했다.**

### why
**AWS**나 **컨테이너화** 개념을 처음 접하는 경우, 또는 <code>Docker</code>를 막 시작하거나 **새로운 애플리케이션을 개발**하는 경우 <code>EB</code>를 통해 컨테이너를 지원하면 좋다.
<code>EB</code>는 간단한 인터페이스를 제공하고, **AWS**에 <code>Docker</code> **컨테이너를 배포**하는 것을 매우 간단하게 진행할 수 있다.

그러나 현재 **수동 배포**를 <code>**ECS**</code> 서비스에 업데이트 하고 있어 이미 **레거시 클러스터**가 존재하기도 하고, <code>Docker</code> **Container Registry** 이력을 유지하는게 좋겠다고 생각했다.
또한, <code>**Github Actions**</code> **워크플로우**도 제로부터 작성해보고 싶었고, 파트별로 **브랜치**를 나눠 **수시로 커밋**을 하다보니 **PR**만 해도 **자동으로 배포되는 프로세스**가 좀 더 편할 것 같았다.

**기능적**으로도 <code>EB</code>와 비교하여 <code>Docker</code> 컨테이너의 **아키텍처** 및 **오케스트레이션**에 대해 더 많은 제어를 제공한다.
**스케줄링**, **CPU 및 메모리 활용**에 있어 **높은 유연성**과 사용자 정의를 제공하여, 다른 **AWS** 서비스와 **통합이 필요한 마이크로 서비스**를 실행하거나 사용자 지정 또는 **관리형 스케줄러**를 사용해 **EC2 온디맨드**, **예약** 또는 **배치 워크로드**를 실행해야 할 때 좋다.

무엇보다 **레거시 코드**를 **컨테이너화** 하고 코드를 다시 작성할 필요 없이 **AWS로 마이그레이션 하려는 경우**에는 <code>**ECS**</code>를 선택해야 쉽게 마이그레이션이 가능하다는 글을 보았다 :)

**이러한 이유에서 ECS를 선택하게 되었다.**

<br>

### Process
일반적으로 아래의 **배포절차**를 거쳐 서비스(개발) **웹 애플리케이션**을 배포한다.

1. <code>ECR</code> 푸쉬 (build Docker, 이미지 태깅)
2. <code>ECS</code> 태스크 개정
3. <code>ECS</code> 신규 태스크 배포
4. <code>CloudFront</code> 캐시 무효화
5. Slack Notification

<br>

### Task definition JSON 세팅
서비스 **태스크 개정**에 필요한 **json**은 태스크를 생성할 때 받을 수 있다.

![](https://velog.velcdn.com/images/hoonlocal/post/966d902e-7d0b-4170-8337-0f3fa012348a/image.png)

해당 **JSON**을 프로젝트 **루트 디렉토리**에 json파일로 세팅하고, 깃액션 **워크플로우 태스크 개정 단계**에서 활용한다.
```yml
# 예시 (dev-task-def.json)

- name: Fill in the new image ID in the Amazon ECS task definition
id: task-def
uses: aws-actions/amazon-ecs-render-task-definition@v1
with:
  task-definition: dev-task-def.json
  container-name: MatsSsoDevContainer
  image: ${{ steps.build-image.outputs.image }}
```

<br>

### 워크플로우
**구글링**과 다른 프로젝트 **레거시 코드**를 참고하여 **.yml** 파일을 작성하였고, 깃허브 **develop** 브랜치에 코드가 **병합**될 때 개발버전 **배포 워크플로우**가 동작하도록 설계했다.
> _to-be_

```yml
name: Deploy Development

on:
  push:
    branches: [develop]

# 해당 Workflow의 하나 이상의 Job 목록
jobs:
  deploy_dev:
    name: Deploy Dev
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16.x]

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3.0.0
        with:
          fetch-depth: 0

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.FE_STAYPIA_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.FE_STAYPIA_AWS_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}   

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Generate dotenv
        run: |
          echo "BASE_URL=https://admin.api.staypia.com/prod" >> .env
          echo "API_KEY=FprmqLkwCoaBclWAOjqLF9gych2AJ0g818jLhQq4" >> .env

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: staypia-admin-web/refactor
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build a docker container and
          # push it to ECR so that it can
          # be deployed to ECS.
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: staypia-admin-web-task.json
          container-name: StaypiaAdminContainer
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: StaypiaAdminRefactor
          cluster: StaypiaAdminWebCluster
          # 기본값 false, 최종확인시 true
          wait-for-service-stability: false

      # CF 캐시 무효화 # cf에서 캐시 설정하지 않도록 변경해서 주석처리
      # - name: Invalidate CloudFront Cache
      #   run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} --paths "/*"

      - name: build result to slack
        uses: 8398a7/action-slack@v3.12.0
        with:
          job_name: 스테이피아어드민 개발버전 배포
          status: ${{job.status}}
          fields: repo,message,commit,author,action,eventName,ref,workflow,job,took
          author_name: Frontend CI
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SLACK_WEBHOOK_URL: ${{ secrets.FE_SLACK_WEBHOOK_URL }}
        if: always()
```

<br>

### 결과 모니터링
배포 **성공유무**에 대한 **피드백**은 **슬랙채널**(fe-github-actions)로 받는다.

<img src="https://velog.velcdn.com/images/hoonlocal/post/9732bee5-6c80-4f18-ae7f-83c65ec013a4/image.png" width="600" />

<br>

## What I Learn
+ **ECS(Elastic Container Service)를 직접 배포하고 시행착오를 겪으면서 ECS에 대한 개념에 보다 더 디테일하게 접근할 수 있었다.**
  - **Docker** 컨테이너를 지원하는 **오케스트레이션** 서비스
  - 가상 머신 **클러스터**를 관리, 확장
  - 해당 VM의 컨테이너를 예약하고 VM **가용성 유지 관리**
  - **AWS Fargate**를 사용하여 컨테이너를 배포 및 관리하므로 서버를 **프로비저닝**할 필요 없음
  - 자체 Amazon VPC에서 컨테이너를 시작하여 **세분화된 보안 제어**를 제공하여 **VPC 보안 그룹** 및 **네트워크 ACL**을 사용할 수 있도록 함
  - IAM을 사용하여 컨테이너가 접근할 수 있는 **서비스와 리소스 결정** 가능
  - **ELB(또는 ALB)**, **CloudWatch**, **ECR** 등과 같은 AWS 서비스를 기본 **통합**으로 사용 가능
  - 클러스터 노드의 **크기와 수**를 지정하고 **자동 크기 조정**을 사용해야 하는지 등을 결정
  - **태스크**에는 함께 시작하고 동시에 종료해야 하는 **그룹**을 지정할 수 있음
  - **다른 AWS 서비스**와 함께 작동하기 위한 특별한 통합 노력이 필요하지 않음

<br>

+ **EB(Elastic Beanstalk)의 특징에 대해 알게 되었다.**
  - 웹 애플리케이션 및 서비스를 **배포하고 확장**하기 위한 서비스
  - **EB**를 이용하면 AWS 리소스를 수동으로 실행할 필요 없이, **Github Actions**과 같은 것을 사용하여 **Docker** 컨테이너 이미지 업로드 가능
  - 애플리케이션을 지원하기 위한 **최신 패치** 및 **업데이트** 제공을 포함한 인프라를 **프로비저닝**하고 기본 플랫폼을 관리하는 **컨테이너 배포**를 처리
  - EB 콘솔을 사용하면 어플리케이션을 **단일 단위로 중지** 및 **시작**하며 관리 가능
  - **오토스케일링** 설정값을 통해 필요할 때에 어플리케이션 **자동 확장** 및 **축소** 가능
  - EB 구축 시 **ALB** 설정으로 **로드밸런싱** 또한 자동으로 처리 가능

<br>

+ **Github Actions 워크플로우를 작성할 수 있게 되었다.**
  - <code>Workflow</code> : 여러 **Job**으로 구성되고, **Event**에 의해 트리거될 수 있는 자동화된 프로세스
  - <code>Event</code> : 특정 브랜치로 **Push**, **Pull Request** 하거나 특정 시간대 반복(**Cron**) 등
  - <code>Job</code> : **Job**은 여러 **Step**으로 구성되고, 다른 **Job**에 의존 관계를 갖거나 독립적으로 병렬 실행도 가능
  - <code>Step</code> : **Task**들의 집합으로, 커맨드를 날리거나 **Action**을 실행할 수 있음
  - <code>Action</code> : **Workflow**의 가장 작은 블럭이고, **Job**을 만들기 위해 **Step**들을 연결할 수 있음
  - <code>Runner</code> : **Github Actions Runner** 애플리케이션이 설치된 머신으로, **Workflow**가 실행될 인스턴스
