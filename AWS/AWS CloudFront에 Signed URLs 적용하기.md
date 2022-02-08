
[미디어파일 S3에서 CloudFront로 이사하기 (feat: CORS)](https://velog.io/@leehaeun0/%EB%AF%B8%EB%94%94%EC%96%B4%ED%8C%8C%EC%9D%BC-S3%EC%97%90%EC%84%9C-CloudFront%EB%A1%9C-%EC%9D%B4%EC%82%AC%ED%95%98%EA%B8%B0-feat-CORS) 글에 이어지는 글입니다.
지난 번엔 이미지 S3에서 CloudFront로 이전하는 방법에 대해 알아보았었고, 이번 글에서는 CloudFront에 Singed URLs 를 적용하는 방법에 대해 알아보려고 합니다.

# Signed URLs
회사에서 사용하는 이미지들은 보안이 필요하기 때문에 아무나 접근할 수 없도록 이미지 url에도 인증 절차가 필요하다.
이럴 때 **`Signed URLs`** 을 사용하면 된다.

> **Signed URL**이란 어떠한 요청을 수행하는 데 필요한 제한된 권한과 시간을 제공하는 URL이다.

Signed URL에는 쿼리스트링에 인증정보, 만료 날짜 및 시간 같은 추가 정보가 포함되므로 **콘텐츠에 대한 액세스를 보다 세부적으로 제어**할 수 있다.

# CloudFront에 Signed URLs 적용하기
CloudFront에서 Signed URLs를 사용하는 방법을 간단히 요약하자면 다음과 같다.

- Signed URLs 인증 절차에 사용할 `공개키`와 `개인키` RSA key pair를 만든다.
- `공개키`를 AWS CloudFront keygroups에 등록하고 해당 CloudFront에 접근제한을 설정하고 keygroups을 연결한다.
- 애플리케이션에서 `개인키`를 사용하여 인증정보가 쿼리에 담겨있는 Signed URLs을 생성한다.
- 해당 Signed URLs로 접근하면 url 쿼리에 등록된 `개인키`가 CloudFront에 등록된 `공개키`와 `복호화`를 성공하여 콘텐츠에 접근이 가능하다.

CloudFront에서 Signed URLs 를 사용하려면 ‘루트 사용자가 aws에서 key pair 생성하도록 하는 방법’과 ‘키 그룹을 생성하는 방법’이 있는데 [aws에선 키 그룹을 생성하는 방법을 권장한다.](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/private-content-trusted-signers.html#choosing-key-groups-or-AWS-accounts) 이 글에서도 키 그룹방식으로 진행할 것이다.

## RSA key pair 생성

1. OpenSSL을 사용하여 길이가 2048비트인 RSA 키 페어를 생성하고 `private_key.pem`이라는 파일에 저장한다.

```bash
openssl genrsa -out private_key.pem 2048
```

1. 이렇게 만들어진 파일은 공개키와 개인키를 모두 포함한다. 다음 명령어를 통해 `private_key.pem`이라는 파일에서 `public_key.pem` 라는 파일로 공개키를 추출한다.

```bash
openssl rsa -pubout -in private_key.pem -out public_key.pem
```

참고문서: [aws 공식 문서](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/private-content-trusted-signers.html#private-content-creating-cloudfront-key-pairs)

## AWS CloudFront 에 공개키 연결

**CloudFront public key 등록**

AWS의 `CloudFront > Public Key > Create public key` 에서 공개키를 등록할 수 있다.

Name 을 지정하고 Key 를 입력하는 곳에 public_key.pem을 등록한다. `cat public_key.pem` 이라는 명령어를 통해 확인한 키의 문자열을 복붙하면 된다.

![](https://images.velog.io/images/leehaeun0/post/0af29eb5-b957-4295-b094-32b439d6b624/image.png)

Create public key를 누르면 공개키 등록이 완료되고 등록된 키를 목록페이지에서 확인해 볼 수 있다.

![](https://images.velog.io/images/leehaeun0/post/6327c8f0-4222-4429-8ff4-cc200a9c31c2/image.png)

**CloudFront Key groups 등록**

그 다음 `CloudFront > Key groups > Create key group` 경로를 통해 키그룹 생성 페이지로 이동한다.

Public keys에서 방금 만든 공개키를 선택하고 키 그룹 생성을 완료한다.

![](https://images.velog.io/images/leehaeun0/post/c7b2b0a9-6247-41d3-8f07-86717e5f337b/image.png)

**해당 CloudFront id에 key groups 연결**

`CloudFront > Distributions > CloudFront id 선택 > origin > edit` 으로 이동 후, Restict viewer assess(뷰어 액세스 제한) 을 yes 로 선택하고 방금 만든 키 그룹을 등록한다.

![](https://images.velog.io/images/leehaeun0/post/0d804a06-123f-4bb6-ba0e-edab98b6bf9d/image.png)

이제 CloudFront에 접근 제한 설정이 완료되었다. 이전에 CloudFront에 접근했던 url 접근하면 다음과 같은 화면이 나오면서 더이상 접근이 안된다. Signed URLs 를 통해서만 접근이 가능하다.

![](https://images.velog.io/images/leehaeun0/post/c88dbecb-977f-4404-a17f-3e65c219fc07/image.png)

## Signed URLs 생성

이제 `Signed URLs` 를 코드상에서 생성해주면 된다.

[AWS 공식문서에서 제공하는 언어별 코드 예제](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/PrivateCFSignatureCodeAndExamples.html)를 참고하면 된다.

우리는 파이썬과 django-storages 라이브러리를 사용하는데 django-storages 최신버전에서는 settings.py에 다음과 같이 설정하면 Signed URLs를 자동으로 생성해 준다. ([문서 링크](https://django-storages.readthedocs.io/en/latest/backends/amazon-S3.html#cloudfront-signed-urls))

```python
AWS_CLOUDFRONT_KEY = os.environ.get('AWS_CLOUDFRONT_KEY',None).encode('ascii')
AWS_CLOUDFRONT_KEY_ID = os.environ.get('AWS_CLOUDFRONT_KEY_ID',None)
```

AWS_CLOUDFRONT_KEY 에는 생성했었던 개인키 `private_key.pem` 파일을 등록해주면 된다.

AWS_CLOUDFRONT_KEY_ID 에는 AWS `CloudFront > Public Key` 에서 등록했었던 key의 id를 등록해주면 된다.

만약 django-storages가 최신버전이 아니라서 위 기능이 지원되지 않는다면 [링크](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/cloudfront.html#generate-a-signed-url-for-amazon-cloudfront)를 참고하여 Signed URLs를 직접 생성하면 된다.

위 방법을 통해 생성된 Signed URLs 주소로 들어가면 접근 제한된 CloudFront에 접근이 가능하다.