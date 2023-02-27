## Issue
maimovie 프로젝트를 단독으로 하고 있는데 기존에 되어있던 환경, 빌드, 배포 등 전반적으로 복잡한게 많아 회사내 공유 용도로 문서화 하려고 한다.

<br>

## Problem
+ 마이무비는 프로젝트는 총 3군데 레포지토리에서 관리되고 있음(마이무비EN 1개, 마이무비KR 2개)
+ 마이무비 국문 버전을 모노레포 방식으로 전환하다가 중단되어서 레포지토리가 2개
  - maimovie : 마이무비(EN), nuxt.js
  - maimovie-ko : 마이무비(KR) 나머지 페이지, nuxt.js
  - maimovie-monorepo : 마이무비(KR) 메인 페이지, next.js
  - 
<br>

## Solution
1. **nvm**으로 노드버전 변경 후 install > 실패

<br>

## What I Learn
