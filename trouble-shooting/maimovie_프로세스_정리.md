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
  - `maimovie-prod-env` : 실서버
+ 데이터는 영문버전 `AWS EFS`, 국문버전 `API`
+ 영문버전과 국문버전에서 사용하는 **데이터구조** 및 **값**들이 다르기 때문에 호환안됨
  - ex) "더글로리"의 영문에서 **series**값은 20136283, 국문에서는 25001439
+ 영문버전은 로컬에서 데이터 확인하려면 **EFS**가 연결되어 있어야 함(연결 및 사용법은 아래 정리)

<br>

## Solution
1. **nvm**으로 노드버전 변경 후 install > 실패

<br>

## What I Learn
