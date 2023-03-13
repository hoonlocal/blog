## Issue
이번에 maimovie(en) AWS EB 서버를 새로 구축하면서 노드환경 버전을 16으로 구축하였다. 그에 맞게 버전을 업그레이드 해야 한다.

<br>

## Problem
+ 서버 환경은 노드 16버전, 프로젝트 환경은 12버전이라 16버전으로 migration 해야 함

<br>

## Solution
1. **nvm**으로 노드버전 변경 후 install > 실패
2. 에러메시지를 보니 `node-sass`, `sass-loader` 패키지가 문제인 듯 하여 노드 **16버전**과 **호환**되는 버전으로 변경 후 다시 시도 > 실패

NodeJS  | Supported node-sass version | Node Module
--------|-----------------------------|------------
Node 19 | 8.0+                        | 111
Node 18 | 8.0+                        | 108
Node 17 | 7.0+, <8.0                  | 102
Node 16 | 6.0+                        | 93
Node 15 | 5.0+, <7.0                  | 88
Node 14 | 4.14+                       | 83
Node 13 | 4.13+, <5.0                 | 79
Node 12 | 4.12+, <8.0                 | 72
Node 11 | 4.10+, <5.0                 | 67
Node 10 | 4.9+, <6.0                  | 64
Node 8  | 4.5.3+, <5.0                | 57
Node <8 | <5.0                        | <57

3. **npm7** 이전에는 **peer dependency** 관련 오류가 있으면 경고만 하고 설치는 됐었는데 이후 부터는 **에러**가 발생하면서 설치가 되지 않는다고 함
4. 무시하고 설치해봤지만 역시나 오류

```
// 의존성 오류 
Could not resolve dependency:
peerOptional node-sass@"^4.0.0" from sass-loader@8.0.2

// 와 함께 두가지 방법을 제안한다.
// 강제로 설치, 또는 의존성 무시하고 설치 위 명령어 뒤에 붙여주면 된다.
this command with --force, or --legacy-peer-deps
```
5. 구글링하여 방법을 찾아봤지만 `sass-loader`와 `webpack`, `webpack`과 `vue` 등을 **다운그레이드**를 통해 호환시켜야 한다고 함
6. 정해진 일정이 있는데 위 방법으로 시도하다간 일정을 못맞출 것 같아 일단 `node-sass`를 안쓰기로 결정
7. `node-sass` 삭제하고 `sass` 설치
8. `node-sass` 사용하지 않으면서 나는 코드 에러 **/deep/** > **::v-deep**으로 변경
9. 추가로 margin-top: **200/30 * 10** 형식 > margin-top: **calc(200 / 300) * 10** 식으로 일괄 변경
10. 마지막으로 **모듈 패키지** 재설치하고 **yarn cache clean**하여 캐시 지우고 배포 > 성공

<br>

## What I Learn
+ 버전때문인지 갑자기 나타나는 Vuex 오류 해결방법을 알게 됨
  - Vuex를 사용할 때 "./"가 Vue 템플릿 루트로 허용되지 않는다는 오류 메시지가 나타났다.
  - 이 오류는 Vuex가 템플릿 옵션을 처리하는 방식의 문제로 인해 발생한다고 한다.
    1. Vuex 스토어에서 템플릿 옵션을 사용하는 대신 렌더링 기능을 사용하여 구성 요소의 HTML을 생성할 수 있다. 이렇게 하면 "./" 오류가 발생하지 않고 구성 요소가 제대로 렌더링될 수 있다.
    2. 또 다른 해결책은 Vuex 스토어를 별도의 Vue 파일로 옮기는 것이다. 이렇게 하면 "./" 오류가 발생하지 않고 템플릿 옵션을 사용할 수 있다.
    3. 하나 더 간단한 해결책은 Vuex가 설치된 module package.json에서 ./를 ./*로 변경해주어도 된다.

a.
```javascript
import { mapState } from 'vuex'

export default {
  computed: {
    ...mapState(['count'])
  },
  render(h) {
    return h('div', `Count: ${this.count}`)
  }
}
```
b.
```javascript
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment(state) {
      state.count++
    }
  },
  actions: {
    increment(context) {
      context.commit('increment')
    }
  },
  getters: {
    count: state => state.count
  }
})

// component.vue
<template>
  <div>
    Count: {{ count }}
  </div>
</template>

<script>
  import store from './store'

  export default {
    computed: {
      count() {
        return store.getters.count
      }
    }
  }
</script>
```
c.
```json
"exports": {
  ".": {
    "module": "./dist/vuex.esm.js",
    "require": "./dist/vuex.common.js",
    "import": "./dist/vuex.mjs"
  },
  "./*": "./*"
},
```
+ node-sass와 sass-loader에 대한 개념 및 옵션들에 대해 부가적으로 학습
+ sass-loader와 webpack의 버전을 맞추는 식으로도 해결할 수 있는 인사이트를 얻게 됨
