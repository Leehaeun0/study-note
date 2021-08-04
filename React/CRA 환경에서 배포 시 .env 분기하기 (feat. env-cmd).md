# 배경

배포환경이 dev 서버와 실서버가 구분되어 있을 때, 사용 하는 환경변수들을 다르게 설정해야 되는 경우가 있다.

예를 들어 Api url 이 prodution 서버는 `example.co.kr` 이고, dev 서버는 `dev.example.co.kr` 일 수가 있는데 이럴때는 env 파일을 따로 분리해서 유동적인 환경변수를 사용해야 한다.

<br/>

# [CRA 를 통한 env 파일의 사용법](https://create-react-app.dev/docs/adding-custom-environment-variables/#what-other-env-files-can-be-used)

**CRA으로 사용할 수 있는 env 파일은 아래와 같다.**

- `.env`: 기본.
- `.env.local`: 로컬 override. 이 파일은 test를 제외한 모든 환경에 대해 로드된다.
- `.env.development`, `.env.test`, `.env.production`: 환경별 설정.
- `.env.development.local`, `.env.test.local`, `.env.production.local`: 환경별 설정의 로컬 override.

<br/>

**이때, script 명령어 별 env 파일 우선순위는 아래와 같다.**

- npm start: `.env.development.local`, `.env.local`, `.env.development`, `.env`
- npm run build: `.env.production.local`, `.env.local`, `.env.production`, `.env`
- npm test: `.env.test.local`, `.env.test`, `.env`

<br/>

여기서 발생하는 문제는 CRA을 사용한 프로젝트에서 env 파일을 사용하는 우선순위는 위와같이 CRA에서 등록한 명령어에 따라 정해져 있으며 이것은 override로 커스터마이징 할 수 가 없다는 것이다... 😱
[(CRA 공식문서)](https://create-react-app.dev/docs/adding-custom-environment-variables/)

![](https://images.velog.io/images/leehaeun0/post/bb4c189b-b4f2-4b16-bf43-89573dec7e77/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-08-04%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%201.02.30.png)

그래서 배포시 build 를 할때에 아래와 같이 build:dev 는 `.env.development` 을 실행하고 싶고 build:prod 는 `.env.production` 을 실행하고 싶은 상황이 와도, build 라는 명령어는 무조건 `.env.production` 를 실행하게 되어있다......

```json
"scripts": {
  	"build:prod": "NODE_ENV=production react-script build",
	"build:dev": "NODE_ENV=development react-script build"
}
```

<br/>

# env-cmd 사용 이유

환경변수를 build 시에 분기할 수 있는 방법이 뭐 없을까 서치해 보다가 [이 깃허브 이슈](https://github.com/facebook/create-react-app/issues/3903)를 보고 [**`env-cmd`**](https://github.com/toddbluhm/env-cmd) 라이브러리를 사용하는 게 해결책이라는 힌트를 얻었다.

[위 이슈 스레드에서 이렇게 답한 사람의 글을 보고 좀더 자세히 이해할 수 있었다.](https://github.com/facebook/create-react-app/issues/3903#issuecomment-403220661)

![](https://images.velog.io/images/leehaeun0/post/5dff78e1-22ee-4f35-ae47-8e514265861c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-08-04%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.16.14.png)

CRA은 내부적으로 **[dotenv](https://github.com/motdotla/dotenv)** 라는 라이브러리를 사용한다.

때문에 `"start": "MY_ENV=123 react-script start"`  이렇게 명령어 줄에 직접 작성하면 CRA 에서 어떤 env 파일을 실행하는 지와 상관 없이 yarn start 명령어로 실행했을 때 `MY_ENV` 의 값은 항상 `123` 이 된다.

다만 좀 더 첨언을 하자면, "CRA을 사용했을 경우" 환경변수 이름은 반드시 REACT\_APP\_ 으로 시작해야만 적용되기 때문에 정확히는 `REACT_APP_MY_ENV` 로 작성해야 아래처럼 제대로 된 결과를 확인할 수 있다.

package.json ⬇️

![](https://images.velog.io/images/leehaeun0/post/f7c07f48-ae07-41cb-abd7-10b3c5c86e6e/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-08-04%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%202.09.16.png)

console.log 결과 ⬇️

![](https://images.velog.io/images/leehaeun0/post/cc69f8ed-8e34-4623-8356-6e3dc6df1df6/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-08-04%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%202.09.24.png)

env-cmd 라이브러리도 위와 동일한 원리로 작동하며, 명령줄에 환경 변수를 모두 나열하는 대신 파일에 환경 변수를 넣을 수 있게 해준다. **따라서 env-cmd 로 지정한 .env 파일의 모든 환경 변수는 cmd 명령줄에서 사용되고, CRA가 사용하는 모든 env 파일보다 우선시 한다. **

<br/>

# env-cmd 사용 방법

**[env-cmd](https://github.com/toddbluhm/env-cmd)** 라이브러리를 설치해 준다.

```bash
yarn add env-cmd
```

root 위치인 /src 파일에 `.env.development` 파일과 `.env.production` 파일을 생성하고 아래와 같이 환경변수를 각각 등록한다.

![](https://images.velog.io/images/leehaeun0/post/f8df3311-a37b-49a7-b7bd-1ba503c143b8/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-08-04%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.38.36.png)

package.json 파일을 아래와 같이 수정해준다.

```json
"scripts": {
	"build:prod": "react-script build",
	"build:dev": "env-cmd -f .env.development react-script build",
	...
}
```

이제 프로젝트 내 파일에서 `process.env.<NAME>` 을 통해 build 스크립트 별 다른 환경변수를 사용할 수 있다.

개인적으로는 Production 서버 여부를 확인하는 코드는 사용 빈도가 높음으로 utils 폴더 파일에 아래와 같이 isProduction 변수를 따로 등록해서 사용하는 것을 선호한다.

```tsx
export const isProduction = process.env.REACT_APP_MODE === 'production';
```

아래는 `build:dev` 명령어를 통해 환경변수를 console.log 한 결과이다.

![](https://images.velog.io/images/leehaeun0/post/b0b6c7b6-2ae2-47cd-a8e5-a984cd63b659/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-08-04%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%204.16.31.png)

![](https://images.velog.io/images/leehaeun0/post/9abad2f9-588d-46c1-ab77-e36971e06210/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-08-04%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%204.15.35.png)

주의할 점은, process.env.NODE_ENV 자체는 보다싶이 production 이고, 변경되지 않았다. 아마도 CRA에서 내부적으로 세팅된 env 파일은 그대로이기 때문인 것 같다.

**대신에 env 파일 안에 등록된 환경 변수들 값은 override 되었다.** 서버 환경은 REACT_APP_MODE 값을 통해 확인하도록 한다.

<br/>

# 같이 볼 수 있는 링크

[CRA .env 지정해서 쓰는법(env-cmd)](https://medium.com/kangtaehun-io-devtory/cra-env-지정해서-쓰는법-env-cmd-27c0dd05a106)

[Vue에서 .env 를 통해 배포 분기하는 방법](https://freevuehub.github.io/devlog-5/)

[node express 에서 .env 를 통해 배포 분기하는 방법](https://gofnrk.tistory.com/116)