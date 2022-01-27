# 배경

나는 회사에서 백오피스 프론트엔드를 개발한다. 백오피스를 사용하시는 실무자 분들이 이미지 파일을 다운로드 받을때 이미지를 하나씩 클릭해서 우클릭 후 다운로드 받는 작업을 하셨었다.

이 과정이 상당히 비효율적이라 생각해서 '이미지 다운로드 버튼', '이미지 전체 다운로드 버튼' 기능을 만드는게 어떨까 내가 의견을 제시했었고 다른분들도 기능의 필요성에 동의해서 개발을 진행하기로 했다. 그리고 여기서부터 모든게 시작되었다..

이미지 다운로드를 하기 위해서는 이미지 url에 [responseType](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/responseType) 을 blob으로 설정한 get 요청보내고 blob 데이터를 받아와야한다.

그런데 기존에 이미지 url을 S3 url로 사용하고 있었고, S3 url에 get 요청을 하면 아래 사진과 같이 `‘Access-Control-Allow-Origin’`이 없다는 `CORS 에러`가 났다.

![](https://images.velog.io/images/leehaeun0/post/bf96a158-ba55-47c7-a601-018fad80716e/image.png)

<br>

# S3 버킷 CORS 설정

해당 CORS 에러를 해결하고자 S3 버킷의 Permissions 탭의 **Cross-origin resource sharing (CORS)** 설정을 다음과 같이 작성 해주었다.

![](https://images.velog.io/images/leehaeun0/post/2d8287c5-d90c-4bf5-b8cd-ccfc46d01573/image.png)

```html
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "GET",
            "HEAD"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": [
            "x-amz-server-side-encryption",
            "x-amz-request-id",
            "x-amz-id-2"
        ],
        "MaxAgeSeconds": 3000
    }
]
```

<br>

# CloudFront를 통해 Origin 설정

그러나.. 이렇게 해도 여전히 CORS 에러가 발생한다. 이 에러는 크롬 브라우저에서만 발생하는 버그였고 다른 브라우저에서는 정상적으로 get요청이 되었다.

위 에러를 해결하고자 검색해보니 나와 같은 에러를 겪고있는 [블로그 글](https://velog.io/@kimsehwan96/S3-CORS-%ED%97%A4%EB%8D%94-%EA%B4%80%EB%A0%A8-%EC%9D%B4%EC%8A%88-%ED%95%B4%EA%B2%B0%EB%B0%A9%EB%B2%95-html2canvas-lottie)이 있었고 정리해보면 다음과 같이 해결했다.

1. `access-control-allow-origin` 이 헤더에 없어서 발생하는 CORS에러니 헤더에 `access-control-allow-origin` 이 어떻게든 들어가도록 해야한다.
2. 요청을 넣을때에 request header 에 `Origin`을 넣으면 response header에 `access-control-allow-origin` 을 넣어 내려보내준다.
3. 그러나 [Orign MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin)에서 설명하다 싶이 Origin은 **[forbidden header name](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name)** 이라서 프로그래밍 방식으로 Origin을 설정할 수가 없다.
4. CloudFront에서는 request header에 Origin을 설정할 수 있다.
5. S3를 바라보는 CloudFront를 생성하고 request header Origin을 설정하니 response header에 `access-control-allow-origin` 이 담겨서 내려왔고 CORS 에러가 해결되었다.

S3에 CloudFront 를 붙인 구조는 다음과 같다. **클라이언트가 S3로 바로 접근하는 것이 아닌 CloudFront(CDN)을 통해서 S3로 접근**하는 것이다.

> 클라이언트 ⇒ CloudFront ⇒ S3

CloudFront를 사용하면 다운로드 에러를 해결할 뿐만이 아니라, **캐쉬**된 데이터인 CloudFront로 접근하는게 S3보다 훨씬 속도가 빠르기 때문에 이미지 url을 CloudFront로 변경하기로 결정했다.

CloudFront의 request header에 Origin을 설정하는 방법은 다음과 같다.

AWS 에서 `CloudFront > Distributions > CloudFront id 선택 > behavior > edit` 이 경로를 통해 접근하면 CloudFront의 request를 설정할 수 있다.

참고로 이 request는 CloudFront가 S3로 보내는 request이다. cloudfront는 중간다리 역할이라서 원본에게 요청을 보내는 request 세팅도 할 수 있고, cloudfront에 접근하는 쪽(주로 클라이언트)에 응답을 하는 Response 세팅도 할 수 있다.

![](https://images.velog.io/images/leehaeun0/post/72427cb0-d03a-4413-a47f-3ac8b625a518/image.png)

aws에서 아주 친절하게도 자주 사용되는 origin request 정책들을 제공해준다. View policy를 통해 aws에서 제공한 CORS-S3Origin 정책을 보면 다음과 같이 생겼다. 보다싶이 headers 에 origin을 설정해주고 있다.

![](https://images.velog.io/images/leehaeun0/post/dc574489-c6cb-4947-9561-0fa0e24d13f5/image.png)

이제 이렇게 하면 아래와 같이 response header에 `access-control-allow-origin` 이 들어갔고 request header 에 `origin`이 들어갔다. Status Code도 200으로 get 요청에 성공했다! 🎉 🎉 🎉
이제 CloudFront 를 통해서 S3에 있는 이미지를 가져올 수 있게 되었고 다운로드도 정상적으로 된다.

![](https://images.velog.io/images/leehaeun0/post/dce90dcf-c241-474e-a9ef-e3b42845929b/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202022-01-19%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2010.57.58.png)

<br>

# S3 접근권한에 CloudFront등록

위 CORS error 해결과는 상관없이 CloudFront를 S3와 연결할때에 짚고 넘어갈 부분이 하나 있다.

`CloudFront > Distributions > CloudFront id 선택 > origin > edit` 에서 bucket access 접근 설정을 할 수 있다. S3를 선택한 후, 버킷정책에서 Yes, update the bucket policy를 선택하고 저장하면 S3의 버킷 정책이 업데이트된다.

![](https://images.velog.io/images/leehaeun0/post/640583d2-bb1a-4d74-a3f0-f068681687f1/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202022-01-19%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2010.55.31.png)

업데이트된 S3 버킷 정책은 `S3 > Bucket > Permissions` 탭에서 확인해볼 수 있다.

Principal을 `“*”` all로 설정해서 아무 경로로나 다 S3에 접근 가능하도록 열어두지 말고, 보안을 생각해서 CloudFront을 통해서만 S3에 접근할 수 있도록 하는것을 추천한다.

![](https://images.velog.io/images/leehaeun0/post/a795278b-2754-4fb9-aa2e-ea7b2aae6e6d/image.png)

<br>

# 아직 남은 일 Signed URLs
이제 CloudFront 를 통해서 이미지를 가져올 수 있게 되었지만, 추가적으로 해줘야 할 작업이 아직 남아 있었다.
회사에서 사용하는 이미지들은 보안이 필요하기 때문에 **`Signed URLs`**을 통해 url 쿼리 문자열에 인증정보를 추가하는 작업이 필요하다. AWS CloudFront에서 Signed URLs 을 사용하는 방법은 다음 글에서 이어 작성하겠다.

