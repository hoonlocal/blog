## Issue
특정 프로젝트의 리포지토리를 옮긴 후 테스트한 첫 배포에 에러가 발생하였다.

![](https://velog.velcdn.com/images/hoonlocal/post/f12fea39-0c19-48ef-97ca-20cdce494763/image.png)

<br>

## Problem
+ 에러메세지를 보니 AWS region 문제인 것 같음
+ 프로젝트 AWS 인증에서 사용되는 키는 회사계정에서 관리되는 전역 공통 환경변수를 활용하는데 확인해보니 region은 등록이 안되어 있음

<img src="https://velog.velcdn.com/images/hoonlocal/post/8de1684b-deef-4f24-844b-81bc8231006b/image.png" width="800" />

<br>

## Solution
1. 해당 **레포지토리** <code>Settings</code> > <code>Security</code> > <code>Secrets and variables</code> > <code>action</code>에 **repository secrets**을 추가해 줌

![](https://velog.velcdn.com/images/hoonlocal/post/4755fa11-288b-4e07-b3bc-7e760b9418eb/image.png)

2. **region**을 **환경변수에 설정**하고 나니 성공적으로 **배포**가 됨

![](https://velog.velcdn.com/images/hoonlocal/post/0625cbc4-9b8e-4000-aa54-de3141bf8630/image.png)

<br>

## What I Learn
+ 깃허브에서 AWS region 인증 오류가 발생하는 경우, 다음과 같은 방법을 통해 해결할 수 있는걸 알게 되었다.
  - AWS 자격 증명 확인
    + AWS 계정에 로그인하여 자격 증명(액세스 키와 비밀 액세스 키)을 확인
    + 이 자격 증명이 정확한지 확인
  - 깃허브 리포지토리 환경 변수 설정
    + 깃허브 리포지토리의 설정에서 'Secrets' 메뉴로 이동
    + AWS 자격 증명 정보를 등록. 이때, 'Name'은 사용할 변수 이름이고, 'Value'는 액세스 키와 비밀 액세스 키를 조합한 문자열
  - Github Actions Workflow 수정
    + Github Actions Workflow 파일(.yml 파일)을 수정하여, AWS 자격 증명 정보가 제대로 전달되도록 함
    + AWS CLI 또는 AWS SDK를 사용하여 AWS 리소스에 액세스하는 경우, 깃허브 액션에서 AWS 자격 증명 정보를 환경 변수로 설정
    + 다음은 AWS 자격 증명 정보를 환경 변수로 설정하는 예시
```yml
name: Example workflow

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Deploy to AWS Elastic Beanstalk
      run: |
        eb init my-application --platform node.js
        eb deploy
```
위 방법을 따라 AWS 자격 증명 정보를 등록하고 Github Actions Workflow 파일을 수정하면, 깃허브에서 AWS region 인증 오류를 해결할 수 있다.
