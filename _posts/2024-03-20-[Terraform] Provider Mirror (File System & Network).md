---
title: "[Terraform] Provider Mirror (File System & Network)"
date: 2024-04-01 17:38:00 +0900
categories: [Terraform]
tags: [iac, terraform]
---

# Provider Mirror
* terraform은 provider mirror 명령어를 제공한다.
* 명령어 실행 시 현재 구성에 필요한 proivder를 로컬 파일 시스템의 디렉터리로 다운로드할 수 있다.
* 미러링 된 디렉토리를 바라볼 수 있도록 terraform cli를 구성하면 인터넷을 통하지 않고 provider를 사용할 수 있다. (explicit installation method configuration)

# File System Mirror
### 1) .tf파일에 required_providers를 명시
hachicorp에서 게시한 공식 AWS 프로바이더인 registry.terraform.io/hashicorp/aws는 줄여서 hashicorp/aws로 사용 가능 (registry.terraform.io hostname은 생략 가능)
```hcl
terraform { 
  required_providers { 
    aws = { 
      source  = "hashicorp/aws" 
      version = "5.12.0" 
    } 
  } 
}
```
### 2) terraform providers mirror 명령어 실행
Mac에서 사용할 경우 platform은 darwin_arm64

```bash
terraform providers mirror -platform=darwin_arm64 ./
```
### 3) 실행 결과
```bash
/home/purrgarr/providers
├── main.tf
└── registry.terraform.io
    ├── getstackhead
    │   └── acme
    │       ├── 1.5.0-patched.json
    │       ├── index.json
    │       └── terraform-provider-acme_1.5.0-patched_linux_amd64.zip
    └── hashicorp
        ├── aws
        │   ├── 3.11.0.json
        │   ├── 3.12.0.json
        │   ├── 5.12.0.json
        │   ├── 5.40.0.json
        │   ├── index.json
        │   ├── terraform-provider-aws_3.11.0_linux_amd64.zip
        │   ├── terraform-provider-aws_3.12.0_linux_amd64.zip
        │   ├── terraform-provider-aws_5.12.0_darwin_arm64.zip
        │   └── terraform-provider-aws_5.40.0_linux_amd64.zip
        └── random
            ├── 2.3.0.json
            ├── index.json
            └── terraform-provider-random_2.3.0_linux_amd64.zip
```
### 4) 로컬에 저장된 프로바이더 사용
```hcl
# ~/.terraformrc
provider_installation {
  filesystem_mirror {
    path = "/home/purrgarr/providers"
  }
}
```
.terraformrc 파일에 프로바이더를 다운로드 받은 경로를 설정해주면 provider 다운로드 시 퍼블릭 테라폼 레지스트리를 바라보는 것이 아닌 설정된 로컬 경로에서 프로바이더를 가져온다.



# Network Mirror
### 1) Provider Network Mirror Protocol
network mirror의 url 설정이 tf-registry.purrgarr.com/providers와 같을 때 몇가지 GET request에 대하여 다음을 만족해야 한다.

1) GET - 사용 가능한 버전 목록
```bash
# GET /:hostname/:namespace/:type/index.json
 
curl 'https://tf-registry.purrgarr.com/providers/registry.terraform.io/hashicorp/random/index.json'
 
# Sample Response
{
  "versions": {
    "2.0.0": {},
    "2.0.1": {}
  }
}
```
2) GET - 사용 가능한 설치 패키지 목록
```bash
# GET /:hostname/:namespace/:type/:version.json
 
curl 'https://tf-registry.purrgarr.com/providers/registry.terraform.io/hashicorp/random/2.0.0.json'
 
# Sample Response
{
  "archives": {
    "darwin_amd64": {
      "url": "terraform-provider-random_2.0.0_darwin_amd64.zip",
      "hashes": [
        "h1:4A07+ZFc2wgJwo8YNlQpr1rVlgUDlxXHhPJciaPY5gs="
      ]
    },
    "linux_amd64": {
      "url": "terraform-provider-random_2.0.0_linux_amd64.zip",
      "hashes": [
        "h1:lCJCxf/LIowc2IGS9TPjWDyXY4nOmdGdfcwwDQCOURQ="
      ]
    }
  }
}
```

### 2) Nginx를 통해 Network Mirror 서버 구성
* GET request의 uri 구조를 보면 File System Mirror의 디렉토리 구조와 동일하다.
* nginx 기본 설정에서 root는 `/usr/share/nginx/html` 디렉토리를 바라본다.
* `/usr/share/nginx/html` 디렉토리에 아래와 같이 File System Mirror와 동일하게 구성한다.
* 구성이 완료되면 network mirror로 사용 가능
```bash
/usr/share/nginx/html/providers
├── main.tf
└── registry.terraform.io
    ├── getstackhead
    │   └── acme
    │       ├── 1.5.0-patched.json
    │       ├── index.json
    │       └── terraform-provider-acme_1.5.0-patched_linux_amd64.zip
    └── hashicorp
        ├── aws
        │   ├── 3.11.0.json
        │   ├── 3.12.0.json
        │   ├── 5.12.0.json
        │   ├── 5.40.0.json
        │   ├── index.json
        │   ├── terraform-provider-aws_3.11.0_linux_amd64.zip
        │   ├── terraform-provider-aws_3.12.0_linux_amd64.zip
        │   ├── terraform-provider-aws_5.12.0_darwin_arm64.zip
        │   └── terraform-provider-aws_5.40.0_linux_amd64.zip
        └── random
            ├── 2.3.0.json
            ├── index.json
            └── terraform-provider-random_2.3.0_linux_amd64.zip
```
### 3) Network Mirror 사용
```hcl
# ~/.terraformrc
provider_installation {
  network_mirror {
    url = "https://tf-registry.purrgarr.com/providers/"
  }
}
```

<br>

Reference
* https://developer.hashicorp.com/terraform/internals/provider-network-mirror-protocol
* https://developer.hashicorp.com/terraform/internals/provider-registry-protocol
* https://servian.dev/terraform-local-providers-and-registry-mirror-configuration-b963117dfffa
