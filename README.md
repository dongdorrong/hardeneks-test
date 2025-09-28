# HardenEKS + Kubent with GitHub Actions (OIDC)

GitHub Actions에서 OIDC로 AWS IAM 역할을 가정해 **HardenEKS** 및 **kube-no-trouble(kubent)** 스캔을 자동화합니다.
워크플로우는 수동 실행(`workflow_dispatch`)을 기본으로 하며, 필요 시 스케줄 실행도 설정할 수 있습니다.
두 도구의 리포트는 GitHub Actions **Artifacts**에 보관됩니다.

---

## 📦 프로젝트 구조

```
.
├── config/
│   ├── config.yaml            # (선택) HardenEKS 규칙 설정
│   └── hardeneks-rbac.yaml    # (선택) 수동 RBAC 사용 시
└── .github/workflows/
    └── hardeneks-kubent.yml   # GitHub Actions 워크플로우
```

> **권장:** 레포트는 저장소에 커밋하지 않습니다. `.gitignore`에 `reports/`를 추가하세요.

```
# .gitignore
reports/
```

---

## ✅ 사전 준비

* EKS 클러스터: `<CLUSTER_NAME>` (리전 `<AWS_REGION>`)
* GitHub OIDC Provider (AWS IAM): `token.actions.githubusercontent.com`
* GitHub Actions용 IAM Role: `<AWS_HARDENEKS_ROLE_ARN>`

IAM Role, Access Entry, Secrets 준비는 HardenEKS 공식 문서를 참고하세요.

---

## 🚀 워크플로우 (hardeneks-kubent.yml)

`.github/workflows/hardeneks-kubent.yml`

```yaml
name: HardenEKS + Kubent Scan

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

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region "${{ secrets.AWS_REGION }}" \
            --name   "${{ secrets.EKS_CLUSTER_NAME }}"

      - name: Allow this runner IP to EKS API
        id: allowapi
        shell: bash
        run: |
          set -euo pipefail
          REGION='${{ secrets.AWS_REGION }}'
          CLUSTER='${{ secrets.EKS_CLUSTER_NAME }}'
          MYIP="$(curl -fsSL ifconfig.me || curl -fsSL https://checkip.amazonaws.com || curl -fsSL https://api.ipify.org)"
          MYIP="$(echo -n "$MYIP" | tr -d '\r\n')"
          MYCIDR="${MYIP}/32"
          CUR_JSON=$(aws eks describe-cluster --region "$REGION" --name "$CLUSTER" \
                        --query 'cluster.resourcesVpcConfig.publicAccessCidrs' --output json)
          NEW_JSON=$(jq --arg ip "$MYCIDR" 'if index($ip) then . else . + [$ip] end' <<<"$CUR_JSON")
          echo "orig=$(printf %s "$CUR_JSON" | base64 -w0)" >> "$GITHUB_OUTPUT"
          if [ "$NEW_JSON" != "$CUR_JSON" ]; then
            aws eks update-cluster-config --region "$REGION" --name "$CLUSTER" \
              --resources-vpc-config publicAccessCidrs="$(jq -r 'join(",")' <<<"$NEW_JSON")"
            aws eks wait cluster-active --region "$REGION" --name "$CLUSTER"
          fi

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Create venv & install HardenEKS
        run: |
          python3 -m venv /tmp/.venv
          source /tmp/.venv/bin/activate
          pip install --upgrade pip
          pip install 'setuptools<81' hardeneks
          echo "/tmp/.venv/bin" >> "$GITHUB_PATH"

      - name: Run HardenEKS (config 조건부 적용)
        run: |
          mkdir -p "$REPORT_DIR/hardeneks"
          ARGS=( \
            --region  "${{ secrets.AWS_REGION }}" \
            --cluster "${{ secrets.EKS_CLUSTER_NAME }}" \
            --export-html "$REPORT_DIR/hardeneks/report.html" \
            --export-json "$REPORT_DIR/hardeneks/report.json" \
            --export-csv  "$REPORT_DIR/hardeneks/report.csv" \
          )
          if [ -s "config/config.yaml" ]; then
            ARGS+=( --config "$(pwd)/config/config.yaml" )
          fi
          hardeneks "${ARGS[@]}"

      - name: Install kubent (latest)
        run: |
          sh -c "$(curl -sSL https://git.io/install-kubent)"
          kubent -v

      - name: Run kubent scan
        run: |
          mkdir -p "$REPORT_DIR/kubent"
          kubent -o json -O "$REPORT_DIR/kubent/kubent.json" || true
          kubent -o csv  -O "$REPORT_DIR/kubent/kubent.csv"  || true
          if jq -e 'length > 0' "$REPORT_DIR/kubent/kubent.json" >/dev/null 2>&1; then
            echo "kubent: deprecated API usage found."
            exit 2
          else
            echo "kubent: no deprecated API usage detected."
          fi

      - name: Upload reports
        uses: actions/upload-artifact@v4
        with:
          name: reports-${{ secrets.EKS_CLUSTER_NAME }}
          path: ${{ env.REPORT_DIR }}/
          retention-days: 14

      - name: Revert EKS API allowlist
        if: always()
        run: |
          REGION='${{ secrets.AWS_REGION }}'
          CLUSTER='${{ secrets.EKS_CLUSTER_NAME }}'
          ORIG_B64='${{ steps.allowapi.outputs.orig }}'
          if [ -n "$ORIG_B64" ]; then
            ORIG_JSON="$(echo "$ORIG_B64" | base64 -d)"
            aws eks update-cluster-config --region "$REGION" --name "$CLUSTER" \
              --resources-vpc-config publicAccessCidrs="$(jq -r 'join(",")' <<<"$ORIG_JSON")"
            aws eks wait cluster-active --region "$REGION" --name "$CLUSTER"
          fi
```

---

## 📑 결과물

* HardenEKS 리포트: `reports/<CLUSTER_NAME>/hardeneks/`
* kubent 리포트: `reports/<CLUSTER_NAME>/kubent/`
* GitHub Actions **Artifacts**에서 다운로드 가능

---

## 🛠️ Troubleshooting

| 증상                          | 원인/대응                                                      |
| --------------------------- | ---------------------------------------------------------- |
| `kubent: command not found` | 설치 스크립트 실행 실패. curl 접근 가능 여부 확인.                           |
| `exit code 2`               | kubent가 deprecated API 발견. 매니페스트 점검 필요.                    |
| HardenEKS RBAC 에러           | Access Entry + `AmazonEKSViewPolicy` 연결 또는 수동 RBAC 바인딩 필요. |

---