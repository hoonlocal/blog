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
+ 깃허브에서 AWS region 인증 오류가 발생하는 경우, 다음과 같은 방법을 통해 해결할 수 있다.
  - AWS 자격
