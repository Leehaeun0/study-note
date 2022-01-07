# 자바스크립트와 비동기

자바스크립트 엔진은 `싱글 스레드`이다. 단 하나의 실행 컨텍스트 스택을 갖는다. 이는 하나의 작업만 처리가 가능하고 동시에 두개 이상의 함수를 병렬적으로 처리할 수 없는 것을 뜻한다.

```jsx
console.log(1);

setTimeout(() => console.log(2), 0);

console.log(3);
```

위와 같은 코드를 실행했을때 콘솔로그가 찍히는 순서는 1, 2, 3 이 아니라 1, 3, 2 이다.

만약 싱글 스레드인 자바스크립트가 1, 2, 3 순차적으로 코드를 실행 한다면 setTimeout 함수가 실행되는동안 다른 코드를 읽지 못하고 블로킹이 된 다음에 `console.log(3)` 가 실행될 것이다. 이런 동작은 성능상 큰 문제가 있다.

따라서 setTimeout 함수같은 경우는 비동기로 동작한다. 비동기 함수는 런타임시 모든 코드가 실행이 되고 난 다음에 실행이 된다. 이것이 setTimeout 을 0ms 로 설정해도 항상 마지막에 동작하는 이유이다.

# Callback

```jsx
function add5(num, callback) {
  console.log(num); // 0, 5, 10, 15
  setTimeout(() => callback(num + 5), 1000);
}

add5(0, function(a) {
  add5(a, function(b) {
    add5(b, function(c) {
      add5(c, function(d) {
        console.log(d); // 20
      })
    })
  })
});
```

위에서 말했다 싶이 비동기 함수는 런타임시 모든 코드가 실행이 되고 난 다음에 실행이 된다.

비동기 함수가 실행되고 난 “다음에” 비동기 처리의 결과물을 바탕으로 코드를 실행하고 싶을때, 아주 예전에는 위와 같이 콜백 방식으로 처리했다.

**콜백 함수**란 위와 같이 함수(`add5`)의 인자에 함수(`callback`)를 넘겨,  해당 함수(`add5`) 안에서 호출되는 함수(`callback`)이다.

add5 함수안에서 callback 함수가 실행되니, add5 함수 안의 코드가 실행되고 난 뒤 callback함수가 실행된다는 “**순서가 보장이 된다**.”

![](https://images.velog.io/images/leehaeun0/post/aacaaf00-aef0-4710-8cb2-279384297abe/ezgif-2-458d35d463.gif)

위 코드를 실행한 결과이다. setTimeout 함수에서 설정한 1초 만큼의 시간 뒤에 차례대로 5씩 더하는 결과를 볼 수 있다. 비동기 처리가 순서대로 잘 보장되어 실행되었다.

하지만 콜백함수의 코드를 보면 바로 문제점을 알 수 있는데, 뎁스가 점점 깊어져 가독성이 매우 안좋다. 이걸 보고 **콜백 헬(callback hell)**이라고 부른다.

# [Promise](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Promise)

```jsx
function add5(num) {
  console.log(num); // 0, 5, 10, 15
  return new Promise(resolve => {
    setTimeout(() => resolve(num + 5), 1000);
  });
}

add5(0)
  .then((a) => add5(a))
  .then((b) => add5(b))
  .then((c) => add5(c))
  .then((d) => {
    console.log(d); // 20
  });
```

위와 같은 문제 때문에 ES6에서 `Promise` 문법이 나왔다.

Promise 는 [생성자 함수](https://ko.javascript.info/constructor-new)이다. 생성자 함수 Promise 를 호출하면 [Promise 인스턴스 객체](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Promise#%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4_%EB%A9%94%EC%84%9C%EB%93%9C)를 리턴한다.

Promise 객체는 [`then`](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Promise/then)이라는 메서드가 있다. 이 then 메서드를 호출할 때 인수로 콜백함수를 넣으면 Promise 에서 resolve로 받아서 비동기 처리를 수행한다. 위 콜백 예시의 callback 인수와 Promise 예시의 resolve 인수가 비슷하게 동작한다.

```

resolve  :  (a) => add5(a)
num + 5  :  a
```

위 처럼 대응된다고 생각하면 좀 더 코드를 쉽게 읽을 수 있을 것 같다.

[then을 계속 체이닝해서 사용](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#chained_promises)할 수 있는데 그 이유는 then 메서드는 또 다시 Promise 객체를 리턴하기 때문에 then을 연속해서 사용할 수 있는 것이다.

이때 전자의 then 메서드의 인수로 들어간 콜백함수(`(a) => add5(a)`)의 return 값(`add5(a)`)이 다음 then 메서드의 콜백함수(`(b) => add5(b)`)의 인자(`b`)로 들어온다. 즉 `add5(a)` 값이 `b`가 된다.

## Promise의 에러 처리

```jsx
function add5(num) {
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(num + 5), 1000);
		if (num === 10) reject('num cannot be 10');
  });
}

add5(0)
  .then((a) => add5(a))
  .then((b) => add5(b))
  .then((c) => add5(c)) // error 발생 위치
  .catch((err) => console.log(err)) // num cannot be 10
  .then((d) => add5(d));
```

reject 함수가 호출되면 Promise 작업이 거절되었다는 것인데 이때 에러를 캐치하는 `catch` 메소드가 호출된다.

then - resolve 관계와 마찬가지로, catch 메소드의 인수로 넣은 콜백함수는 reject로 호출된다. 따라서 reject 함수의 인수로 오는 값(`'num cannot be 10'`)이 catch 인수인 콜백함수의 인자(`err`) 위치로 들어온다.

에러처리를 바로 하지 않고 다음 then코드를 진행해도 상관없다면 catch를 맨 마지막에 사용할 수 도 있다.

```jsx
add5(0)
  .then((a) => add5(a))
  .then((b) => add5(b))
  .then((c) => add5(c)) // error 발생 위치
  .then((d) => add5(d))
  .catch((err) => console.log(err));
```

사실 then은 두개의 인수를 받을 수 있는데 첫번째 인수로 resovle 함수가 오고 두번째 인수의 위치에는 reject 함수가 실행된다. 하지만 이것 역시 가독성이 좋지 않아서 주로 catch 문으로 에러처리를 하는 편이다.

```jsx
Promise.resolve()
  .then(() => {
    throw new Error('으악!');
  })
  .then(() => {
    console.log('실행되지 않는 코드');
  }, (error) => {
    console.error('onRejected 함수가 실행됨: ' + error.message);
  });
```

기존 `try-catch`를 이용해서도 예외 처리가 가능하지만 자바스크립트에서는 Promise의 catch를 사용하라는 warning message를 출력한다.

참고로 덧붙이자면 아래의 문법은 전부 자바스크립트에서 같이 쓰일 수 있는 문법이다.

```jsx
add5(0).then(add5);
add5(0).then((a) => add5(a));
add5(0).then((a) => { return add5(a) });
add5(0).then(function(c) { return add5(c) });
```

# Async Await

```jsx
function add5(num) {
  console.log(num); // 0, 5, 10, 15
  return new Promise(resolve => {
    setTimeout(() => resolve(num + 5), 1000);
  });
}

async function print() {
  const a = await add5(0);
  const b = await add5(a);
  const c = await add5(b);
  const d = await add5(c);
  console.log(d); // 20
}

print();
```

`async await` 문법은 Promise 보다 나중인 ECMAScript 2017 에 나온 문법이다.

async await 문법은 비동기 코드를 거의 동기 코드 작성하듯이 사용할 수 있어서 가독성이 가장 좋다.

await 키워드 뒤에는 Promise 객체가 온다. await 뒤에오는 Promise 의 상태가 pending일 동안 기다리고 나서 완료된 뒤에 다음 코드를 실행한다. await 를 쓸때는 async를 반드시 써야하고 async 가 붙은 print 함수는 Promise 객체를 return한다.

## Async Await의 에러 처리

```jsx
function add5(num) {
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(num + 5), 1000);
    if (num === 10) reject('num cannot be 10');
  });
}

async function print() {
  try {
    const a = await add5(0);
    const b = await add5(a);
    const c = await add5(b);
    const d = await add5(c);
  } catch (err) {
    console.log(err); // num cannot be 10
  }
}

print();
```

에러 처리에 흔히 사용하는 방법인 `try-catch` 를 사용하여 에러를 처리를 할 수 있다.

```jsx
const a = await add5(0).catch((err) => console.log(err));
const b = await add5(a).catch((err) => console.log(err));
```

add5 함수가 Promise 객체를 return 하기 때문에 함수에 바로 `catch` 문을 이어서 사용할 수도 있지만 실무에서 일반적으로는 try catch 문을 사용하는 편인 것 같다.

# 참고 자료

[자바스크립트 비동기통신 Callback, Promise, Async/Await 이해하기](https://techlog.io/Javascript/General/%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8-%EB%B9%84%EB%8F%99%EA%B8%B0%ED%86%B5%EC%8B%A0-callback-promise-async-await-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0/)
[Mdn Using Promises](https://developer.mozilla.org/ko/docs/Web/JavaScript/Guide/Using_promises)