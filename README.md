# HardenEKS + Kubent with GitHub Actions (OIDC)

GitHub Actions에서 OIDC로 AWS IAM 역할을 가정해 **HardenEKS** 및 **kube-no-trouble(kubent)** 스캔을 자동화합니다.
수동 실행(`workflow_dispatch`) 및 스케줄 실행을 지원하며, 두 도구의 리포트는 워크플로우 아티팩트로 보관합니다.

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

IAM Role과 Access Entry, Secrets 준비는 [HardenEKS README](./README.md)와 동일합니다.

---

## 🚀 워크플로우(hardeneks-kubent.yml)

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

      # HardenEKS 실행
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Create venv & install HardenEKS
        run: |
          python3 -m venv /tmp/.venv
          source /tmp/.venv/bin/activate
          pip install --upgrade pip
          pip install hardeneks
          echo "/tmp/.venv/bin" >> "$GITHUB_PATH"

      - name: Run HardenEKS
        run: |
          mkdir -p "$REPORT_DIR/hardeneks"
          hardeneks \
            --region "${{ secrets.AWS_REGION }}" \
            --cluster "${{ secrets.EKS_CLUSTER_NAME }}" \
            --config config/config.yaml \
            --export-html "$REPORT_DIR/hardeneks/report.html" \
            --export-json "$REPORT_DIR/hardeneks/report.json" \
            --export-csv  "$REPORT_DIR/hardeneks/report.csv"

      # Kubent 실행 (항상 최신 설치)
      - name: Install kubent (latest)
        run: |
          sh -c "$(curl -sSL https://git.io/install-kubent)"
          kubent -v

      - name: Run kubent scan
        run: |
          mkdir -p "$REPORT_DIR/kubent"
          kubent -o json -O "$REPORT_DIR/kubent/kubent.json" || true
          kubent -o csv  -O "$REPORT_DIR/kubent/kubent.csv"  || true

          # deprecated API 발견 시 워크플로우 실패 처리
          if jq -e 'length > 0' "$REPORT_DIR/kubent/kubent.json" >/dev/null 2>&1; then
            echo "kubent: deprecated API usage found."
            exit 2
          else
            echo "kubent: no deprecated API usage detected."
          fi

      - name: Upload combined reports
        uses: actions/upload-artifact@v4
        with:
          name: reports-${{ secrets.EKS_CLUSTER_NAME }}
          path: ${{ env.REPORT_DIR }}/
          retention-days: 14
```

---

## 📑 결과물

* HardenEKS 리포트: `reports/<CLUSTER_NAME>/hardeneks/`
* kubent 리포트: `reports/<CLUSTER_NAME>/kubent/`
* 두 결과는 GitHub Actions **Artifacts**에서 다운로드 가능

---

## 🛠️ Troubleshooting

| 증상                          | 원인/대응                                                      |
| --------------------------- | ---------------------------------------------------------- |
| `kubent: command not found` | 설치 스크립트 실행 실패. 런너 환경에서 curl 접근 가능한지 확인.                    |
| `exit code 2`               | kubent가 deprecated API를 발견. 클러스터 매니페스트 점검 필요.              |
| HardenEKS RBAC 에러           | Access Entry + `AmazonEKSViewPolicy` 연결 또는 수동 RBAC 바인딩 필요. |