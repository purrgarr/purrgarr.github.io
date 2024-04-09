---
title: "[ClusterAPI] Management Cluster에 cluster api provider aws 설치 및 IAM Role 적용"
date: 2024-03-28 17:38:00 +0900
categories: [Kubernetes, ClusterAPI]
tags: [kubernetes, k8s, clusterapi, eks, aws]
---

# 명령어 설치
* 먼저 클라이언트 명령어를 설치한다. (mac 기준)
```bash
brew install clusterctl
brew install clusterawsadm
```

# Cluster API Provider AWS 설치
> Cluster API를 설치할 쿠버네티스 클러스터(Management Cluster)는 준비되어 있다고 가정
{: .prompt-info }

### 시스템 환경변수 설정
```bash
# Cluster API를 설치할 클러스터(Management Cluster)의 kubeconfig
export KUBECONFIG=/Users/purrgarr/.kube/config
# controller가 사용하는 작업증명
export AWS_ACCESS_KEY_ID="ASIAWVU2TIV5WNAAB..."
export AWS_SECRET_ACCESS_KEY="GM94VndABCD0cWc..."
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjEPj..."
export AWS_REGION=ap-northeast-2
# 위 작업증명을 기반으로 AWS 프로필 인코딩
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
```

`AWS_B64ENCODED_CREDENTIALS` 값은 Secret 데이터로 배포되고 controller에서 사용한다.


### ClusterAPI 설치
* 아래 명령어를 통해 cluster api를 설치 한다.
* cluster api가 설치된 클러스터는 Management Cluster로서 동작하게 된다.
```bash
clusterctl init --infrastructure aws
```

### Feature 활성화를 위한 추가 시스템 환경변수

* `EXP_MACHINE_POOL` : MachinePool 사용을 위해 활성화
* `CAPA_EKS_IAM` : 클러스터에 미리 생성한 IAM Role을 붙이기 위해 활성화
<br>_이 외에도 더 존재_

```bash
# 이미 init 수행했으면 Feature 적용 전 delete 수행
clusterctl delete --infrastructure aws --bootstrap=kubeadm --core=cluster-api

# Enable Feature
export EXP_MACHINE_POOL=true
export CAPA_EKS_IAM=true
clusterctl init --infrastructure aws
```

> `clusterctl delete` 하지 않고 `init`만 하면 controller 파드를 삭제해도 다시 뜰때 적용되지 않음. feature 적용 여부는 `k describe po -A | grep feature-gates`로 확인 가능
{: .prompt-info }

# Controller가 사용하는 권한
### 현재 자격증명 확인
* Cluster API Provider AWS Controller(capa-controller)가 사용하는 자격증명은 아래 명령어로 확인 가능하다.

```bash
$ clusterawsadm controller print-credentials --namespace=capa-system 
[default]
aws_access_key_id = ASIAWVU2TIV5WNAAB...
aws_secret_access_key = GM94VndABCD0cWc...
region = ap-northeast-2

aws_session_token = IQoJb3JpZ2luX2VjEPj...
```
* 자격증명 내용은 앞에서 `AWS_B64ENCODED_CREDENTIALS`변수에 인코딩 되어 들어간 값이다.
* 해당 자격증명은 임시 자격증명으로 `aws_session_token`이 만료되면 더이상 사용이 불가능하다.


# IAM Role을 사용하도록 Management Cluster 구성
> AccessKey 및 SecretAccessKey를 영구적으로 발급받아 사용할수도 있지만 키 탈취 등 보안위협이 존재하기 때문에 IAM User보다는 `IAM Role` 기반으로 자격증명을 구성하는 것이 바람직하다.
{: .prompt-warning}

### IAM Role 생성
Cluster API Provider AWS Controller가 사용할 IAM Role 생성

1. AWSIAMConfiguration
 * awsiamconfig.yaml 파일을 아래와 같이 생성 (AWS_ACCOUNT_ID, AWS_REGION, OIDC_PROVIDER_ID 값 변경)
```yaml
apiVersion: bootstrap.aws.infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSIAMConfiguration
spec:
  clusterAPIControllers:
    disabled: false
    trustStatements:
    - Action:
      - "sts:AssumeRoleWithWebIdentity"
      Effect: "Allow"
      Principal:
        Federated:
        - "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/oidc.eks.${AWS_REGION}.amazonaws.com/id/${OIDC_PROVIDER_ID}"
      Condition:
        "ForAnyValue:StringEquals":
          "oidc.eks.${AWS_REGION}.amazonaws.com/id/${OIDC_PROVIDER_ID}:sub":
            - system:serviceaccount:capa-system:capa-controller-manager
            - system:serviceaccount:capa-eks-control-plane-system:capa-eks-control-plane-controller-manager # Include if also using EKS

```

2. IAM Role 생성
 * awsiamconfig.yaml 파일을 config로 아래 명령어를 실행하면 필요한 IAM Role을 자동으로 생성해준다.

    ```bash
    clusterawsadm bootstrap iam create-cloudformation-stack --config awsiamconfig.yaml
    ```

    생성되는 role 목록 다음과 같다.
    * control-plane.cluster-api-provider-aws.sigs.k8s.io (for ec2)
    * controllers.cluster-api-provider-aws.sigs.k8s.io (for ec2)
    * eks-controlplane.cluster-api-provider-aws.sigs.k8s.io	(for eks)
    * nodes.cluster-api-provider-aws.sigs.k8s.io (for ec2)

  * 위 IAM Role 목록 중 `controllers.cluster-api-provider-aws.sigs.k8s.io` 사용. <br>
    추후에 KMS 암호화를 사용하는 EKS를 관리하기 위해서는 아래 policy도 Role에 추가 필요.
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": [
                    "kms:Encrypt",
                    "kms:Decrypt",
                    "kms:ReEncrypt*",
                    "kms:GenerateDataKey*",
                    "kms:DescribeKey",
                    "kms:CreateGrant"
                ],
                "Resource": "*"
            }
        ]
    }
    ```

### credentials 정보 삭제
앞에서 controller에 적용되었던 임시 자격증명을 제거하는 과정

```bash
# credential 정보 삭제
clusterawsadm controller zero-credentials --namespace=capa-system
# controller 재배포하여 적용(다음 단계에서 재설치 진행할거라서 굳이 지금 안해도 됨)
clusterawsadm controller rollout-controller --namespace=capa-system
```
위 명령어 실행 후에는 `clusterawsadm controller print-credentials --namespace=capa-system` 결과가 `Credentials are zeroed`로 출력

### IAM Role을 사용하도록 reinit

```bash
# 삭제
clusterctl delete --infrastructure=aws

# AWS_B64ENCODED_CREDENTIALS가 없으면 명령어 에러 발생. 
export AWS_B64ENCODED_CREDENTIALS=Cg== # 빈 값
# clusterawsadm bootstrap iam 명령어를 통해 생성된 IAM Role
export AWS_CONTROLLER_IAM_ROLE=arn:aws:iam::458811983227:role/controllers.cluster-api-provider-aws.sigs.k8s.io
clusterctl init --infrastructure=aws
```

* init이 완료되면 이제 Cluster API Provider AWS Controller는 IAM Role 기반으로 동작하게 된다.
* `capa-controller` pod 로그를 확인하여 권한이 제대로 동작하는지 확인.


<br>

Reference
* https://cluster-api-aws.sigs.k8s.io/getting-started
* https://cluster-api-aws.sigs.k8s.io/topics/iam-permissions
* https://cluster-api-aws.sigs.k8s.io/topics/using-iam-roles-in-mgmt-cluster
* https://cluster-api-aws.sigs.k8s.io/topics/specify-management-iam-role.html
* https://cluster-api-aws.sigs.k8s.io/topics/using-clusterawsadm-to-fulfill-prerequisites