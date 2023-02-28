## Issue
maimovie 프로젝트를 혼자 하고 있는데 기존에 되어있던 환경, 빌드, 배포 등 전반적으로 복잡한게 많아 회사내 공유 용도로 문서화 하려고 한다.

<br>

## Problem
+ 마이무비 프로젝트는 총 3개의 레포지토리에서 관리되고 있음
  - 마이무비 영문 1개(**maimovie**)
  - 마이무비 국문 2개(**maimovie-monorepo**, **maimovie-ko**)
+ 마이무비 국문버전을 **모노레포** 방식으로 점진적으로 전환(**nuxt** > **next**)하고 있어 레포지토리가 2개
+ 마이무비 국문 중 main, notice, privacy, terms 페이지만 next.js(maimovie-monorepo), 나머지 페이지는 nuxt.js(maimovie-ko)
+ 현재 프로세스에서는 수정요청이 들어오면 영문, 국문(2개) 각각 수정하고, 따로 배포해야함
+ 배포는 영문버전 `AWS EB`, 국문버전 `AWS ECS`
+ 영문버전 **EB**에 들어가보면 애플리케이션 환경이 많은데 그 중 2개의 환경에서만 관리되고 있음
  - `maimovie-release-env` : 개발서버
  - `maimovie-prod-env` : 서비스서버
+ 데이터는 영문버전 `AWS EFS`, 국문버전 `API`
+ 영문버전과 국문버전에서 사용하는 **데이터구조** 및 **값**들이 다르기 때문에 호환안됨
  - ex) "더글로리"의 영문에서 **series**값은 20136283, 국문에서는 25001439
+ 영문버전은 로컬에서 데이터 확인하려면 **EFS**가 연결되어 있어야 함(연결 및 사용법은 아래 정리)
+ Module Rendered 컴포넌트 방식 사용(데이터가 있어야 모튤 노출)

<br>

## Solution
### 마이무비 영문
+ 개발환경 : Nuxt.js(vue)
+ 배포타입 : EB(maimovie-release-env, maimovie-prod-env)
+ 데이터 : EFS
+ 레포지토리 : maimovie
+ 개발서버 : https://release.maimovie.com
+ 서비스서버 : https://maimovie.com
+ dev 배포 : `yarn deploy:dev` 또는 직접 EB에 업로드
+ service 배포 : `yarn deploy:prod` 또는 직접 EB에 업로드
#### EFS
영문버전을 로컬에서 띄었을때 데이터가 제대로 나오지 않는다. AWS EFS를 로컬에 직접 연결시켜야 데이터를 불러올 수 있다. 로컬이 아닌 서버(개발, 서비스)에서는 백엔드 개발자가 EB와 EFS를 연결시켜 두었으므로 별도로 다른 작업을 하지 않아도 데이터가 잘 나온다. 간단한 작업이라면 작업 후 개발서버에서 확인해도 되지만 보다 큰 작업이라면 로컬에서 확인하면서 작업해야하니 EFS를 연결하여 사용한다. EFS는 Profile(/movie, /people) 페이지에서 사용된다.
1. 로컬에서 EFS 사용하려면 macFuse와 sshfs 설치  
+ `brew cask install osxfuse` or https://osxfuse.github.io
+ `brew install sshfs` or https://macappstore.org/sshfs/
2. 설치 후 관련 백엔드 개발자(현재 김진용님)에게 마이무비 영문버전 EFS 권한 요청
3. 요청하면 아래와 같은 펨키를 받음
```
// 예시
-----BEGIN RSA PRIVATE KEY-----
HJapazI4+Mnrj1Q/2aEp9HrvfP73x/9Xrx0mKKP6I326D9BBRHen849JxZmOXRcX
rYeb73VKrsMkpbCJxb0edi3jjXf54z6ZNx0ySrARdI1ZRBTQ2RSpK7W4lXNXd7h9
1ZSkZgwWv/s+UwbtWz1/j3WtXd+F6Jj3jzUvD5rMYjIu4KyoKKIFSt1j2W6Mk+Ur
ksL/+vHX08+/WwxU/pdDIUBsHrWIBz8KhftsMwZCpAN5SyA+oPCIVqlfYBVhbslJ
7w4wv2ECgYEA8tJ+z3RpqFFC5BRaXIaVMg2EGdTDf+uORQpipDE2lR24DDLeTYdc
jW+gPoFy+OCQalL098b0IhIhlj7mVW4UhakrjO1dMjmcYwTvqxFYSwxIoIaApyIt
800JTTcjsd4fmvYbgSTmHkPlm0AzC0K6DfdJAYiXHgrsJ7AZkHqzbPMCgYEAk+nh
9zFgtMYyZnIG3titN2C/GW08YfPqcRgTl4wMmvHiUdSBQLIdHQXmRfpFRHgqJIqW
JkfYUb/JJ0wW/Ufy9ShWESroCsR8B7Mn4vAVGU3R/oFY9gzET+kAfJAjy9AcBU4E
gKzFcYP0dFaw76O4FC6woqkeImLCFNjkLqKyW0kCgYBMS7dsl7dbG61Y3MxHpkHa
-----END RSA PRIVATE KEY-----
```
4. 먼저 위 펨키는 내비두고 터미널에서 ~/.ssh/config 파일에 들어가서 아래와 같이 작성 후 저장
```
Host enmaimovieEFS
  HostName 52.53.155.131
  User {아이디}
  IdentityFile ~/.ssh/maimovie-{아이디}-us-west-1-221221.pem
```
5. 이 후 ~/.ssh에 maimovie-{아이디}-us-west-1-221221.pem 파일을 만들고 아까 발급받은 펨키를 넣고 저장
6. 여기까지 했으면 EFS를 연결하기 위한 펨키 환경설정은 완료되었고, EFS를 불러올 때 마운트, 해제할 때 언마운트 해야 함
7. 마운트
```
sshfs hyanghoon@52.53.155.131:/mnt/maimovie-efs-mount/ {로컬경로} -o IdentityFile=~/.ssh/{pemkey}
```
```
// 예시
sshfs hyanghoon@52.53.155.131:/mnt/maimovie-efs-mount/ "/Users/mycelebs/Desktop/project/maimovie/maimovie-efs" -o IdentityFile=~/.ssh/maimovie-hyanghoon-us-west-1-221221.pem
```
8. 언마운트
```
umount hyanghoon@52.53.155.131:/mnt/maimovie-efs-mount/
```
9. 마운트 로컬경로는 프로젝트에서 jsonFileReader.js 파일의 path.resolve('../maimovie/maimovie-efs')와 맞아야 함
#### env
개발, 서비스 API 수정할 때
1. .env 파일
```
DEV_API=true // dev
DEV_API=false // prod
```
2. jsonFileReader.js 파일
```
isDev = true // dev
isDev = false // prod
```

### 마이무비 국문 - main, notice, provacy, terms
+ 개발환경 : Next.js(react)
+ 배포타입 : ECS(배포자동화)
+ 데이터 : API
+ 레포지토리 : maimovie-monorepo(apps/maimovie_kr)
+ 개발서버 : https://release.ko.maimovie.com
+ 서비스서버 : https://ko.maimovie.com
+ dev 배포 : develop 브랜치에 push시 dev 배포
+ service 배포 : main 브랜치에 push 후 release note 작성 시 배포

### 마이무비 국문 - 나머지(프로필 정보가 있는 페이지들)
+ 개발환경 : Nuxt.js(vue)
+ 배포타입 : ECS(배포자동화)
+ 데이터 : API
+ 레포지토리 : maimovie-ko
+ 개발서버 : https://release.ko.maimovie.com/movie/20094415
+ 서비스서버 : https://ko.maimovie.com/movie/20094415
+ dev 배포 : develop 브랜치에 push시 dev 배포
+ service 배포 : main 브랜치에 push 후 release note 작성 시 배포

<br>

## What I Learn
