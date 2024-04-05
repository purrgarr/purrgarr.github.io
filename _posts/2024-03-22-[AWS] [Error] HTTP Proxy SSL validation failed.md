---
title: "[AWS] [Error] HTTP Proxy SSL validation failed"
date: 2024-03-22 17:38:00 +0900
categories: [AWS]
tags: [aws, troubleshooting, error, http_proxy]
---

# 문제 상황
* HTTP 프록시를 통해 `aws cli`를 사용하기 위해 시스템 환경변수 `HTTP_PROXY`, `HTTPS_PROXY` 설정
* `aws sso login`시 SSL validation 에러 발생
```bash
$ aws sso login

SSL validation failed for https://oidc.ap-northeast-2.amazonaws.com/device_authorization [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self-signed certificate in certificate chain (_ssl.c:1006)
```

<br>

# 해결 방법

* 에러 메시지를 보면 `https://oidc.ap-northeast-2.amazonaws.com/device_authorization`에 대한 인증서 문제인 것을 알 수 있다.
* 먼저 프록시를 통하지 않고 인터넷을 통해 curl 명령을 실행하여 로컬 CA 파일 확보 -> `/etc/ssl/cert.pem`
```bash
$  curl -v   https://oidc.ap-northeast-2.amazonaws.com/device_authorization

*   Trying 3.38.86.105:443...
* Connected to oidc.ap-northeast-2.amazonaws.com (3.38.86.105) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/cert.pem
...중략...
```
* 확보한 CA 파일의 경로를 시스템 환경변수 `AWS_CA_BUNDLE`에 설정
```bash
export AWS_CA_BUNDLE=/etc/ssl/cert.pem
```