# Issue
신규개발이 거의 없던 회사내 레거시 프로젝트 신규 모듈 개발요청이 들어왔다. 개발 후 빌드 > 배포 하려 했으나 에러 발생

<br>

# Problem
+ 소스코드 내 배포 프로세스 확인해봤더니 AWS EB(Elastic Beanstalk) 사용
+ package.json에 작성된대로 yarn deploy:dev 돌려봤으나 실패

<br>

# Solution
1. 백엔드개발자에게 해당 프로젝트 AWS IAM 계정을 받음
2. EB에 들어가서 애플리케이션 환경 파악(dev, patch, prod 서버로 구성되어 있었음)
3. node12 버전으로 환경세팅이 되어있음(해당 프로젝트는 14로 구성)
4. 테스트를 위해 eb deploy 방식이 아닌 알집으로 직접 업로드 해봄 > 실패(__MAXOSX 등 문제)
5. 에러 로그 및 .ebextensions를 보니 AWS EFS 연결이 안되어 있는 것 같음
6. EB에서 EFS에 연결하는 설정을 백엔드개발자에게 요청 > 실패(VPN 등 연결 문제)
7. 오래된 프로젝트라 히스토리를 정확히 알고있는 개발자가 없어 EB를 새로 구축
8. node 환경을 16버전으로 재구성
9. 그에맞게 package도 12 > 16으로 migration
10. eb init > eb create > eb deploy 순서로 배포 성공

<br>

# What I Learn
+ AWS EB 명령어, 사용법, 배포 프로세스에 대한 개념 및 실무 경험
+ dev, patch, prod로 서버를 나누는 이유에 대해 파악
+ 새 버전을 별도의 환경에 배포한 후, 두 환경의 CNAME을 바꿔 트래픽을 새 버전으로 즉시 리디렉션하는 블루/그린 무중단 배포에 대해 알게 됨
+ AWS EFS의 실무 활용 학습
+ 노드 버전을 migration 하면서 나타나는 문제들 해결(node-sass, sass-loader 등)
