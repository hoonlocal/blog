## Issue
디자인 디테일을 자주 확인해야 할 때 빠른 테스트를 위해 수시로 개발서버에 배포하기도 한다. 그럴때마다 시간이 오래걸려 불편했던 점이 있어 배포시간을 단축할 방법을 알아보았다.

## Problem
+ 해당 프로젝트는 Github Actions를 사용
+ 워크플로우를 통해 ECS로 배포

## Solution
1. 워크플로우에서 **배포시간**을 **단축**할 수 있는 방법을 구글링
2. 아래와 같이 **작업 안정성 확인 비활성화**(wait-for-service-stability: false)
```bash
- name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@de0132cf8cdedb79975c6d42b77eb7ea193cf28e
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: false. # <--- default is true
```
```
무중단 배포할때
+ ecs 드레인하고 종료되는 것까지 기다린다: true
+ ecs 드레인을 기다리지 않는다: false
```
3. 테스트 해보니 이 프로젝트의 경우 **30~40%**(17분 -> 6분) 가량 **시간 단축**

## What I Leand
+ 배포를 하면 기존 서버에서 새 서버로 트래픽을 옮김
+ 기본 설정인 `wait-for-service-stability: true` 일 때는 기존 사용자들(기존 서버가 처리중)의 트래픽을 새 서버에 안정적, 점진적으로 이관
+ `wait-for-service-stability: false` 일 때는 기존서버를 멈추고 새 서버에 트래픽을 쏠리게해 상대적으로 빠르게 처리
+ 따라서, 개발서버에서만 사용하는 것이 좋음
