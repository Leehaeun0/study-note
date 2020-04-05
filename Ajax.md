1. #### JSON.stringify메소드를 통해 자바스크립트의 객체를 JSON형식의 문자열로 변환해서 서버로 보낸다.

   ```js
   const JSON형식문자열변수 = JSON.stringify( 객체변수 );
   ```

   </br>

   

2. #### JSON 이라는 문자열 형태로 클라이언트에서 서버로, 서버에서 클라이언트로 데이터를 주고 받는다.

   JSON은 일반 텍스트 포맷보다 효과적인 데이터 구조화가 가능하며 [XML](https://ko.wikipedia.org/wiki/XML) 포맷보다 가볍고 사용하기 간편하며 가독성도 좋다.

   </br>

   

3. #### JSON.parse메소드를 통해 서버에서 받은 JSON형식으로 받은 문자열을 자바스크립트의 객체로 바꾼다. (배열 또한 변환 가능)

```js
const 객체변수 = JSON.parse( JSON형식문자열변수 );
```

</br>

