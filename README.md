# HardenEKS Scan (Local & CI)

HardenEKS로 Amazon EKS 클러스터를 점검하는 최소 예시입니다. 모든 식별자는 **변수**로만 사용합니다.

## 구조
```
config/
config.yaml            # 선택: 규칙/무시 설정 (실행 시 --config로 지정)
hardeneks-rbac.yaml    # ClusterRole(+Binding)
reports/                 # 스캔 결과(커밋 금지)
```

## 준비물
- AWS CLI v2, kubectl, Python 3.11+
- (권장) 리포지토리 비공개
- 실행 주체(IAM 사용자 또는 AssumeRole 대상 역할)에 최소 IAM 권한:
```
  {
    "Version": "2012-10-17",
    "Statement": [
      {"Effect":"Allow","Action":"eks:ListClusters","Resource":"*"},
      {"Effect":"Allow","Action":"eks:DescribeCluster","Resource":"*"},
      {"Effect":"Allow","Action":"ecr:DescribeRepositories","Resource":"*"},
      {"Effect":"Allow","Action":"inspector2:BatchGetAccountStatus","Resource":"*"},
      {"Effect":"Allow","Action":"ec2:DescribeFlowLogs","Resource":"*"},
      {"Effect":"Allow","Action":"ec2:DescribeInstances","Resource":"*"}
    ]
  }
```

## 변수 예시
```
export REGION=<REGION>
export CLUSTER=<CLUSTER_NAME>
export PRINCIPAL_ARN=arn:aws:iam::<ACCOUNT_ID>:<user|role>/<NAME>
export GROUP=hardeneks
```

## 1) Access Entry (IAM 주체 ↔ Kubernetes 그룹)

```
aws eks create-access-entry \
  --cluster-name "$CLUSTER" \
  --principal-arn "$PRINCIPAL_ARN" \
  --type STANDARD \
  --kubernetes-groups "$GROUP" \
|| aws eks update-access-entry \
  --cluster-name "$CLUSTER" \
  --principal-arn "$PRINCIPAL_ARN" \
  --kubernetes-groups "$GROUP"

aws eks describe-access-entry \
  --cluster-name "$CLUSTER" \
  --principal-arn "$PRINCIPAL_ARN" \
  --query 'accessEntry.kubernetesGroups'
```

## 2) RBAC 적용

`config/hardeneks-rbac.yaml`(ClusterRole `hardeneks-runner` + ClusterRoleBinding `hardeneks-binding`)을 적용합니다.

```
kubectl apply -f config/hardeneks-rbac.yaml
kubectl auth can-i list pods -A   # yes면 권한 연결 OK
```

## 3) kubeconfig 생성

```
aws eks update-kubeconfig --region "$REGION" --name "$CLUSTER"
```

## 4) HardenEKS 설치 & 실행

권장: 특정 버전 고정.

```
python -m pip install --upgrade pip
pip install 'hardeneks==0.11.2'
```

실행:

```
mkdir -p reports/"$CLUSTER"
hardeneks \
  --region "$REGION" \
  --cluster "$CLUSTER" \
  --config config/config.yaml \        # 설정 사용 시
  --export-html reports/"$CLUSTER"/report.html \
  --export-json reports/"$CLUSTER"/report.json \
  --export-csv  reports/"$CLUSTER"/report.csv
```

## (선택) GitHub Actions 요약

* OIDC로 IAM 역할 가정 → `aws eks update-kubeconfig` → `pip install hardeneks==0.11.2` → `hardeneks ... --config ... --export-*`
* 결과는 `actions/upload-artifact`로 보관(리포지토리에 커밋하지 않음)

## 정리(Cleanup)

```bash
# 그룹 비우기
aws eks update-access-entry --cluster-name "$CLUSTER" --principal-arn "$PRINCIPAL_ARN" --cli-input-json '{
  "clusterName":"'"$CLUSTER"'",
  "principalArn":"'"$PRINCIPAL_ARN"'",
  "kubernetesGroups":[]
}'

# 또는 Access Entry 삭제
aws eks delete-access-entry --cluster-name "$CLUSTER" --principal-arn "$PRINCIPAL_ARN"
```

## 트러블슈팅

* `forbidden` 또는 `no` 응답: 그룹명(`$GROUP`) ↔ ClusterRoleBinding `subjects.name` 일치 확인
* 설정이 적용되지 않음: `--config config/config.yaml` 플래그 확인
* 시작 경고: `pkg_resources` 경고는 무해, `NoneType...` 이슈 시 `hardeneks==0.11.2` 고정 권장

## 레퍼런스
* https://github.com/aws-samples/hardeneks
* https://aws.amazon.com/ko/blogs/tech/hardeneks-validating-best-practices-for-amazon-eks-clusters-programmatically-kr
