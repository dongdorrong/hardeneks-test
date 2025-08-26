# HardenEKS with GitHub Actions (OIDC)

GitHub Actions에서 OIDC로 AWS IAM 역할을 가정해 **HardenEKS** 스캔을 자동화합니다.  
수동 실행(`workflow_dispatch`) 및 스케줄 실행을 지원하며, 레포트는 워크플로우 아티팩트로 보관합니다.

---

## 📦 프로젝트 구조

```
.
├── config/
│   ├── config.yaml            # (선택) 하드닝 규칙 설정
│   └── hardeneks-rbac.yaml    # (선택) 수동 RBAC 사용 시
└── .github/workflows/
    └── hardeneks.yml          # GitHub Actions 워크플로우
```

> **권장:** 레포트는 저장소에 커밋하지 않습니다. `.gitignore`에 `reports/`를 추가하세요.

```
# .gitignore
reports/
```

---

## ✅ 사전 준비

- EKS 클러스터: `<CLUSTER_NAME>` (리전 `<AWS_REGION>`)
- GitHub OIDC Provider (AWS IAM): `token.actions.githubusercontent.com`
- GitHub Actions용 IAM Role: `<AWS_HARDENEKS_ROLE_ARN>`

### 1) GitHub OIDC Provider (없다면 생성)

- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

### 2) GitHub Actions 전용 IAM Role

**신뢰 정책(Trust policy)** — 리포지토리 단위 제한 권장:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": [
          "repo:<REPO_OWNER>/<REPO_NAME>:*"
        ]
      }
    }
  }]
}
```

**권한 정책(Inline/Managed)**

- 필수: `eks:DescribeCluster`
- (선택) 퍼블릭 엔드포인트 접근제어 CIDR를 **임시 허용/원복**하려면: `eks:UpdateClusterConfig`, `eks:DescribeUpdate`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["eks:DescribeCluster"], "Resource": "*" },
    { "Effect": "Allow", "Action": ["eks:UpdateClusterConfig","eks:DescribeUpdate"], "Resource": "arn:aws:eks:*:<ACCOUNT_ID>:cluster/<CLUSTER_NAME>" }
  ]
}
```

### 3) 클러스터 접근 권한(Access Entry → RBAC)

- **권장:** 해당 IAM Role을 EKS **Access Entry**로 등록하고 **Managed Access Policy** `AmazonEKSViewPolicy`를 연결 (read-only).  
  HardenEKS가 필요한 조회 권한을 얻습니다.
- (대안) `config/hardeneks-rbac.yaml`의 ClusterRole/ClusterRoleBinding을 이용해 수동 RBAC 매핑. Access Entry의 `kubernetesGroups`에 바인딩된 그룹을 연결하세요.

### 4) 저장소 Secrets 설정

`Settings → Secrets and variables → Actions → New repository secret`

| Name | Example |
|---|---|
| `AWS_HARDENEKS_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>` |
| `AWS_REGION` | `ap-northeast-2` |
| `EKS_CLUSTER_NAME` | `<CLUSTER_NAME>` |

(변수로 관리 시 `${{ vars.* }}`로 변경)

---

## 🚀 워크플로우(hardeneks.yml)

`.github/workflows/hardeneks.yml`

```yaml
name: HardenEKS Scan

on:
  workflow_dispatch: {}
  # schedule:
  #   - cron: '0 18 * * *'   # (옵션) 매일 03:00 KST

permissions:
  id-token: write
  contents: read

env:
  REPORT_DIR: reports/${{ secrets.EKS_CLUSTER_NAME }}

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_HARDENEKS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}

      # (옵션) 퍼블릭 API 엔드포인트 접근제어 중이면: 현재 런너 IP를 임시 허용
      # - name: Allow this runner IP to EKS API (temp)
      #   id: allowlist
      #   shell: bash
      #   run: |
      #     set -euo pipefail
      #     REGION="${{ secrets.AWS_REGION }}"
      #     CLUSTER="${{ secrets.EKS_CLUSTER_NAME }}"
      #     MYIP="$(curl -fsS ipconfig.kr || curl -fsS https://checkip.amazonaws.com || curl -fsS https://api.ipify.org)"
      #     [[ "$MYIP" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] || { echo "Failed to detect runner IP"; exit 1; }
      #     CIDR="${MYIP}/32"
      #
      #     CUR=$(aws eks describe-cluster --region "$REGION" --name "$CLUSTER" \
      #           --query 'cluster.resourcesVpcConfig.publicAccessCidrs' --output json)
      #     NEW=$(jq -r --arg ip "$CIDR" 'if index($ip) then . else . + [$ip] end' <<<"$CUR")
      #     if [[ "$CUR" != "$NEW" ]]; then
      #       aws eks update-cluster-config --region "$REGION" --name "$CLUSTER" \
      #         --resources-vpc-config publicAccessCidrs="$(jq -r 'join(\",\")' <<<"$NEW")"
      #       aws eks wait cluster-active --region "$REGION" --name "$CLUSTER"
      #     fi
      #     echo "cidr=$CIDR" >> "$GITHUB_OUTPUT"

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region "${{ secrets.AWS_REGION }}" \
            --name   "${{ secrets.EKS_CLUSTER_NAME }}"

      - name: Sanity check RBAC
        run: |
          kubectl auth can-i get serviceaccounts -n kube-system
          kubectl auth can-i list pods -A

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Create venv & install HardenEKS
        run: |
          python3 -m venv /tmp/.venv
          source /tmp/.venv/bin/activate
          python -m pip install --upgrade pip
          # 필요시 경고 억제:  pip install 'setuptools<81'
          pip install 'hardeneks==0.11.2'
          echo "/tmp/.venv/bin" >> "$GITHUB_PATH"

      - name: Run HardenEKS
        run: |
          mkdir -p "$REPORT_DIR"
          CFG=""
          [[ -f config/config.yaml ]] && CFG="--config config/config.yaml"
          hardeneks \
            --region "${{ secrets.AWS_REGION }}" \
            --cluster "${{ secrets.EKS_CLUSTER_NAME }}" \
            $CFG \
            --export-html "$REPORT_DIR/report.html" \
            --export-json "$REPORT_DIR/report.json" \
            --export-csv  "$REPORT_DIR/report.csv"

      - name: Upload report artifacts
        uses: actions/upload-artifact@v4
        with:
          name: hardeneks-report-${{ secrets.EKS_CLUSTER_NAME }}
          path: ${{ env.REPORT_DIR }}/
          retention-days: 14
          if-no-files-found: error

      # (옵션) 임시 허용 CIDR 원복
      # - name: Revert allowlist
      #   if: always() && steps.allowlist.outputs.cidr
      #   shell: bash
      #   run: |
      #     set -euo pipefail
      #     REGION="${{ secrets.AWS_REGION }}"
      #     CLUSTER="${{ secrets.EKS_CLUSTER_NAME }}"
      #     CIDR="${{ steps.allowlist.outputs.cidr }}"
      #     CUR=$(aws eks describe-cluster --region "$REGION" --name "$CLUSTER" \
      #           --query 'cluster.resourcesVpcConfig.publicAccessCidrs' --output json)
      #     NEW=$(jq -r --arg ip "$CIDR" 'map(select(. != $ip))' <<<"$CUR")
      #     if [[ "$CUR" != "$NEW" ]]; then
      #       aws eks update-cluster-config --region "$REGION" --name "$CLUSTER" \
      #         --resources-vpc-config publicAccessCidrs="$(jq -r 'join(\",\")' <<<"$NEW")"
      #       aws eks wait cluster-active --region "$REGION" --name "$CLUSTER"
```

---

## 🧪 로컬 실행(참고)

```bash
python3 -m venv /tmp/.venv && source /tmp/.venv/bin/activate
pip install -U pip hardeneks
hardeneks \
  --region <AWS_REGION> \
  --cluster <CLUSTER_NAME> \
  --config config/config.yaml \
  --export-html reports/<CLUSTER_NAME>/report.html \
  --export-json reports/<CLUSTER_NAME>/report.json \
  --export-csv  reports/<CLUSTER_NAME>/report.csv
```

---

## 🛠️ Troubleshooting

| 증상 | 원인/대응 |
|---|---|
| `Error: Input required and not supplied: aws-region` | `AWS_REGION`가 Secrets/Variables에 없음. 이름을 정확히 확인. |
| `403 Forbidden ... serviceaccounts "aws-node"` | 클러스터 RBAC 미매핑. **Access Entry + AmazonEKSViewPolicy**(권장) 또는 수동 RBAC 바인딩 적용. |
| `AccessDeniedException: UpdateClusterConfig` | (옵션) 임시 CIDR 허용 단계를 사용할 때 필요한 IAM 권한(`eks:UpdateClusterConfig`, `eks:DescribeUpdate`) 누락. |
| 퍼블릭 엔드포인트 접근제어 활성화로 API 연결 실패 | (옵션) 워크플로우의 “Allow this runner IP …” 단계 사용 → 스캔 후 원복. |
| `pkg_resources is deprecated` 경고 | 동작 영향 없음. 조용히 하려면 `pip install 'setuptools<81'` 추가. |

---

## 🔒 보안 메모

- OIDC 신뢰 정책은 **특정 리포지토리**로 범위를 제한하세요.
- EKS 권한은 **조회 전용(AmazonEKSViewPolicy)** 를 기본으로, 필요한 옵션 단계만 최소 권한으로 추가하세요.
- 레포트는 저장소에 커밋하지 않고 **아티팩트**로만 보관합니다.