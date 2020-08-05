## HTTP

하이퍼텍스트 전송 프로토콜(HTTP)은 HTML과 같은 하이퍼미디어 문서를 전송하기위한 애플리케이션 레이어 프로토콜입니다.   
일반적으로 안정적인 전송 레이어로 UDP와 달리 메세지를 잃지 않는 프로토콜인 TCP/IP 레이어를 기반으로 사용 합니다.   

| method	| route |
|-----|----|
| GET |	/resource |
| GET	| /resource/:id |
| POST	| /resource |
| PUT	| /resource/:id |
| PATCH	|/resource/:id |
| DELETE	| /resource/:id |

### form 태그와 HTTP 통신

form 태그는 http메소드 중에서 GET, POST 두가지만 사용하는 태그이다.

## REST

“Representational State Transfer” 의 약자   
자원을 이름(자원의 표현)으로 구분하여 해당 자원의 상태(정보)를 주고 받는 모든 것을 의미한다.   
즉, 자원(resource)의 표현(representation) 에 의한 상태 전달