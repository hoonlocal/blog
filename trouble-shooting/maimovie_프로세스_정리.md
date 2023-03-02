## Issue
maimovie 프로젝트를 혼자 맡고 있는데 기존에 되어있던 환경, 빌드, 배포 등 전반적으로 복잡한게 많아 회사내 공유 용도로 문서화 하려고 한다.

<br>

## Problem
+ **마이무비 프로젝트**는 **총 3개의 리포지토리**에서 관리되고 있음
  - 마이무비 영문 1개(`maimovie`)
  - 마이무비 국문 2개(`monorepo-maimovie`, `maimovie-ko`)
+ 국문버전을 **모노레포** 방식으로 점진적으로 전환(**nuxt.js** > **next.js**)하고 있어 리포지토리가 2개
+ 국문 중 **main**, **notice**, **privacy**, **terms** 페이지만 **Next.js**(monorepo-maimovie), 나머지 페이지는 **Nuxt.js**(maimovie-ko)
+ 국문(**Nuxt.js**) 소스코드를 **로컬**에서 띄우면 나오는 **첫페이지**는 **에러** 나는게 정상(메인페이지는 **Next.js** 페이지에 있으므로 여기선 **movie/20094415**와 같이 특정 프로필을 붙여 접속)
+ 현재 **프로세스**에서는 수정요청이 들어오면 영문, 국문(2개) **각각 수정**하고, **따로 배포**해야함
+ 배포는 영문버전 `AWS EB`, 국문버전 `AWS ECS`
+ 영문버전 **EB**에 들어가보면 애플리케이션 환경이 많은데 그 중 2개의 환경에서만 관리되고 있음
  - `maimovie-release-env` : 개발서버
  - `maimovie-prod-env` : 서비스서버
+ 데이터는 영문버전 `AWS EFS`, 국문버전 `API`
+ 영문버전과 국문버전에서 사용하는 **데이터구조** 및 **값**들이 다르기 때문에 호환안됨
  - ex) "더글로리"의 영문에서 **series**값은 20136283, 국문에서는 25001439
+ 영문버전은 **로컬**에서 **데이터** 확인하려면 `EFS`가 연결되어 있어야 함(연결 및 사용법은 아래 정리)
+ **Module Rendered 컴포넌트 방식** 사용(데이터가 있어야 모듈 노출)

<br>

## Solution
### 🖥 마이무비 영문
+ **개발환경** : Nuxt.js(vue)
+ **배포타입** : EB(maimovie-release-env, maimovie-prod-env)
+ **데이터** : EFS
+ **리포지토리** : maimovie
+ **개발서버** : https://release.maimovie.com
+ **서비스서버** : https://maimovie.com
+ **dev 배포** : `yarn deploy:dev` 또는 직접 EB에 업로드
+ **service 배포** : `yarn deploy:prod` 또는 직접 EB에 업로드
#### 💡 EFS
영문버전을 로컬에서 띄웠을때 데이터가 제대로 나오지 않는다. AWS EFS를 로컬에 직접 연결시켜야 데이터를 불러올 수 있다.

로컬이 아닌 서버(개발, 서비스)에서는 백엔드 개발자가 EB와 EFS를 연결시켜 두었으므로 별도로 다른 작업을 하지 않아도 데이터가 잘 나온다. 간단한 작업이라면 작업 후 개발서버에서 확인해도 되지만 보다 큰 작업이라면 로컬에서 확인하면서 작업해야하니 EFS를 연결하여 사용한다.

EFS는 Profile(/movie, /people) 페이지에서 사용된다.
1. **로컬에서 EFS 사용하려면 macFuse와 sshfs 설치**  
+ `brew cask install osxfuse` or https://osxfuse.github.io
+ `brew install sshfs` or https://macappstore.org/sshfs/
2. **설치 후 관련 백엔드 개발자(현재 김진용님)에게 마이무비 영문버전 EFS 권한 요청**
3. **요청하면 아래와 같은 펨키를 받음**
```
// 예시
-----BEGIN RSA PRIVATE KEY-----
jfljbDFGDFGDFGDTT65654645645EDB654654/dfbDBGTYYf5635665456dfdFgg
DFBDFbdkfnbkdbdDFBdkn5gn43kn4gker4654645klfndlflknhfgfgGGFGflkdn
4580457dkrvnSVr454SVrgr45fljlvnnDVRGDfddFDfDFsdfgfg4534sdv4s5zp0
-----END RSA PRIVATE KEY-----
```
4. **먼저 위 펨키는 내비두고 터미널에서 ~/.ssh/config 파일에 들어가서 아래와 같이 작성 후 저장**
```
Host enmaimovieEFS
  HostName 52.53.155.131
  User {아이디}
  IdentityFile ~/.ssh/maimovie-{아이디}-us-west-1-221221.pem
```
5. **이 후 ~/.ssh에 maimovie-{아이디}-us-west-1-221221.pem 파일을 만들고 아까 발급받은 펨키를 넣고 저장**
6. **여기까지 했으면 EFS를 연결하기 위한 펨키 환경설정은 완료되었고, EFS를 불러올 때 마운트, 해제할 때 언마운트 해야 함**
7. **프로젝트에서 jsonFileReader.js 파일의 path.resolve('..maimovie-efs')와 맞는 경로에 maimovie-efs 폴더 생성**
8. **마운트**
```
sshfs {아이디}@52.53.155.131:/mnt/maimovie-efs-mount/ {로컬경로} -o IdentityFile=~/.ssh/{pemkey}
```
```
// 예시
sshfs hyanghoon@52.53.155.131:/mnt/maimovie-efs-mount/ "/Users/mycelebs/Desktop/project/maimovie/maimovie-efs" -o IdentityFile=~/.ssh/maimovie-hyanghoon-us-west-1-221221.pem
```
9. **언마운트**
```
umount {아이디}@52.53.155.131:/mnt/maimovie-efs-mount/
```
#### 💡 env
개발, 서비스 API 수정할 때
1. **.env 파일**
```
DEV_API=true // dev
DEV_API=false // prod
```
2. **jsonFileReader.js 파일**
```
isDev = true // dev
isDev = false // prod
```
#### 💡 배포
+ **eb deploy 방식**
  - eb init으로 .elasticbeanstalk 초기화 설정
  - eb create로 애플리케이션 환경 세팅
  - eb deploy로 배포
  - 아래와 같이 개발서버, 서비스서버 배포를 package.json에 설정해둠
  - `"deploy:dev": "yarn build && eb deploy maimovie-release-env --region us-west-1 --profile maimovie",`
  - `"deploy:prod": "yarn build && eb deploy maimovie-prod-env --region us-west-1 --profile maimovie",`

+ **직접 업로드 방식**
  - 위 방식이 안되면 아래 순서로 직접 업로드
  - 배포 전 yarn install(node_modules 설치되어 있으면 생략)
  - 이후 yarn build
  - build 완료 후 node_modules 삭제
  - EFS 연결하며 만들었던 maimovie-efs 폴더도 삭제
  - 압축하기 전 프로젝트 폴더 내에서 숨김파일 보기 설정
  - 프로젝트의 최상위 폴더(maimovie)를 제외한 안의 내용물만 드래그해서 압축
  - 압축한 파일은 EB에 직접 업로드 함(개발서버는 `maimovie-release-env`, 서비스서버는 `maimovie-prod-env`)
  - 혹시 맥에서 배포가 안되면 [__MACOSX 제거](https://asecurity.dev/entry/Mac-Zip-%ED%8C%8C%EC%9D%BC%EC%97%90%EC%84%9C-MACOSX-DSStore-%EC%A0%9C%EA%B1%B0) 참고

+ **기존 EB에 배포된 서비스가 갑자기 동작하지 않는 경우**
  - 로그 확인해보면 npm install 시에 git:// 프로토콜 오류로 배포가 정상적으로 되지 않을때 해결방법
  - package-lock.json 파일을 생성하고 git:// -> https:// 로 수정해서 배포하면 됨
```
1. elastic beanstalk에서 현재 실행되고 있는 버전의 압축파일 다운로드 및 압축풀기 (환경을 선택하고 좌측 애플리케이션 버전 클릭)

2. yarn.lock package-lock.json 간 변환 프로그램 설치
> yarn global add synp

3. package-lock.json 파일 생성
> yarn
> synp --source-file yarn.lock

4. package-lock.json 파일 수정
에디터로 package-lock.json 파일을 열어서 git:// -> https:// 로 수정한다.

5. 배포
build 이후에 압축하여 배포 (.nuxt 폴더 등 히든파일 및 폴더 확인)
```

### 🖥 마이무비 국문 - main, notice, provacy, terms
+ **개발환경** : Next.js(react)
+ **배포타입** : ECS(배포자동화)
+ **데이터** : API
+ **리포지토리** : monorepo-maimovie(apps/maimovie_kr)
+ **개발서버** : https://release.ko.maimovie.com
+ **서비스서버** : https://ko.maimovie.com
+ **dev 배포** : develop 브랜치에 push시 dev 배포
+ **service 배포** : main 브랜치에 push 후 release note 작성 시 배포

### 🖥 마이무비 국문 - 나머지(프로필 정보가 있는 페이지들)
+ **개발환경** : Nuxt.js(vue)
+ **배포타입** : ECS(배포자동화)
+ **데이터** : API
+ **리포지토리** : maimovie-ko
+ **개발서버** : https://release.ko.maimovie.com/movie/20094415 ("정이" 프로필 예시)
+ **서비스서버** : https://ko.maimovie.com/movie/20094415 ("정이" 프로필 예시)
+ **dev 배포** : develop 브랜치에 push시 dev 배포
+ **service 배포** : main 브랜치에 push 후 release note 작성 시 배포

<br>

## What I Learn
+ 해당 프로젝트는 적합한 모노레포 방식은 아니지만, 모노레포 방식의 장단점에 대해 알게 됨
  - 장점
    + 코드 공유 : 여러 프론트엔드 프로젝트를 단일 저장소에 저장 리포지토리를 사용하면 프로젝트 간에 코드를 공유하기가 더 쉬워진다. 이는 개발 주기를 단축하고 버그를 줄일 수 있다.
    + 단순화된 종속성 관리 : 모든 프로젝트가 단일 저장소에 있으므로 종속성을 관리하고 각 프로젝트가 동일한 버전의 공유 패키지를 사용하고 있다.
    + 일관된 도구 및 구성 : 단일 리포지토리를 사용하면 모든 프로젝트가 동일한 도구 및 구성을 사용하도록 하는 것이 더 쉽다. 이는 일관성을 유지하고 구성 오류를 줄이는 데 도움이 될 수 있다.
    + 코드 가시성 향상 : 한 프로젝트의 변경 사항이 어떤 영향을 미치는지 더 쉽게 확인할 수 있다. 이렇게 하면 잠재적인 문제를 식별하고 코드 중복을 줄이는 데 도움이 될 수 있다.
  - 단점
    + 복잡성 증가 : 여러 프론트엔드를 관리하는 단일 리포지토리의 프로젝트는 개별적으로 관리하는 것보다 더 복잡할 수 있다. 이는 더 복잡한 빌드 프로세스, 더 느린 빌드 및 더 어려운 디버깅으로 이어질 수 있다.
    + 더 큰 코드베이스 : 특히 여러 팀이 작업하는 경우 단일 저장소는 시간이 지남에 따라 커지고 다루기 어려워질 수 있다. 이로 인해 빌드 시간이 느려지고 개발 환경이 더 어려워질 수 있다.
    + 충돌 가능성 : 여러 개발자가 동일한 저장소에서 작업하는 경우 두 명 이상의 개발자가 작업할 때 충돌이 발생할 수 있다. 이는 관리하기 어려울 수 있으며 병합 충돌로 이어질 수 있다.
    + 위험 증가 : 단일 리포지토리에 여러 프론트엔드 프로젝트를 저장하면 버그가 발생할 위험이 증가한다. 또는 한 프로젝트의 문제가 다른 프로젝트에 영향을 미칠 수 있다. 이로 인해 테스트 요구 사항이 증가하고 디버깅이 더 어려워질 수 있다.
  - 요약
    + 전반적으로 프론트엔드 단일 저장소 접근 방식은 코드 공유, 종속성 관리 및 일관성 측면에서 상당한 이점을 제공할 수 있다. 그러나 복잡성 증가, 더 큰 코드베이스, 잠재적인 충돌 및 위험 증가가 수반될 수도 있다. 모든 개발 접근 방식과 마찬가지로 프론트엔드 모노레포 저장소를 사용할지 여부를 결정하기 전에 장단점을 신중하게 따져보는 것이 중요하다.
+ EFS의 사용이유와 펨키를 받아 연결하는 방법들을 배움
  - EFS 사용이유
    + 공유 파일 스토리지 : Amazon EFS를 사용하면 여러 Amazon EC2 인스턴스에서 파일을 공유할 수 있으므로 동일한 파일에 쉽게 액세스할 수 있다. 다른 인스턴스에서 파일을 복사할 필요가 없다.
    + 확장성 : Amazon EFS는 용량을 프로비저닝할 필요 없이 애플리케이션의 요구 사항에 맞게 자동으로 확장되도록 설계되었다. 이를 통해 증가하는 데이터 양과 증가하는 워크로드를 쉽게 처리할 수 있다.
    + 고가용성 및 내구성 : Amazon EFS는 여러 가용 영역에 데이터를 저장하여 고가용성과 내구성을 제공하도록 설계되었다. 이렇게 하면 데이터를 항상 사용할 수 있고 장애로부터 보호할 수 있다.
    + 비용 효율성 : Amazon EFS는 종량제 방식의 파일 스토리지를 위한 비용 효율적인 솔루션이다. 사용한 스토리지 및 처리량에 대해서만 비용을 지불할 수 있는 모델이다.
    + 다른 AWS 서비스와 손쉬운 통합 : Amazon EFS는 다음과 같은 다른 AWS 서비스와 쉽게 통합된다. Amazon EC2, AWS Lambda 및 Amazon Elastic Kubernetes Service(Amazon EKS)를 통해 기존 AWS 인프라의 일부로 쉽게 사용할 수 있다.
    + 유연한 액세스 : Amazon EFS는 네트워크 파일 시스템(NFS)v4 프로토콜 및 Amazon EFS 파일 동기화를 포함한 다양한 액세스 옵션을 통해 다양한 애플리케이션 및 환경과 쉽게 통합할 수 있다.
+ 모듈 랜더링 컴포넌트 방식에 대해 학습
+ AWS Elastic Beanstalk에 배포하는 방법과 그 과정에 나타나는 에러들을 처리하고 해결할 수 있게 됨
+ 복합한 레거시 코드를 추적하며 선임 개발자의 좋은 코드 스타일을 익힘
