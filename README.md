# sample-app-with-tekton

OpenShift 上で **Tekton** が CI（ソース取得 → コードビルド → UT → 静的スキャン → イメージビルド → イメージスキャン）を順に実行するデモ用リポジトリです。アプリは Maven で `target/sample.war` を生成し、`ContainerFile` で Tomcat イメージに載せます。

## Tekton CI（メイン）

### 前提

- `oc login` 済み、対象 namespace を選択
- OpenShift Pipelines（Tekton）がインストール済み（`oc get csv -n openshift-operators | grep pipelines` など）
- イメージ push 先（内部レジストリなど）と、必要なら pull/push 用 Secret

### ディレクトリ

```
tekton/
├── tasks/           # git-clone, code-build, unit-test, static-scan, image-build, image-scan
├── pipelines/     # sample-app-ci
├── pipelineruns/  # 手動起動例
└── rbac/          # ServiceAccount pipeline
```

### 適用順

```bash
export NS=your-namespace
oc project "$NS"

oc apply -f tekton/rbac/pipeline-sa.yaml
oc apply -f tekton/tasks/
oc apply -f tekton/pipelines/

# Task 定義を更新したあと、古い PipelineRun だけでは clone イメージは変わらない。再作成すること

# buildah 用: クラスタポリシーで anyuid が必要な場合あり（下記「よくある失敗」）
# oc adm policy add-scc-to-user anyuid -z pipeline -n "$NS"

# PipelineRun の git-url / image を編集してから
oc create -f tekton/pipelineruns/sample-app-ci-run.yaml
```

`tekton/pipelineruns/sample-app-ci-run.yaml` の `git-url`・`image`・（任意）`docker-credentials` Secret を環境に合わせて変更してください。内部レジストリでは `tls-verify: "false"` の例を入れています。

レジストリ認証がある場合:

```bash
oc create secret docker-registry pipeline-push-secret \
  --docker-server=image-registry.openshift-image-registry.svc:5000 \
  --docker-username="$(oc whoami)" \
  --docker-password="$(oc whoami -t)" \
  -n "$NS"
# PipelineRun の docker-credentials コメントを外して secretName を合わせる
```

### CI の見方

```bash
tkn pipelinerun list -n "$NS"
tkn pipelinerun logs -f -n "$NS" -l app.kubernetes.io/part-of=sample-app-ci

oc get pipelinerun,taskrun -n "$NS"
oc describe pipelinerun <name> -n "$NS"
tkn taskrun list -n "$NS"
```

成功条件: TaskRun が **fetch-repository → code-build → unit-test → static-scan → image-build → image-scan** の順ですべて `Succeeded`（静的・イメージスキャンは dummy で常に PASS）。

### ローカルビルド（参考）

```bash
mvn -B package -DskipTests && mvn -B test
podman build -f ContainerFile -t sample-app:local .
```

### よくある失敗

| 現象 | 対処 |
|------|------|
| UT 失敗 | `tkn taskrun logs <unit-test-run> -n "$NS"` で Surefire を確認 |
| Maven 依存の取得失敗 | クラスタから Maven Central へ出られるか、プロキシ設定 |
| `buildah` / push 失敗 | `pipeline` SA にレジストリ push 権限・`docker-credentials` Secret・SCC（`anyuid` 等） |
| `fetch-repository` の ImagePullBackOff | `registry.redhat.io` は未認証だと pull 不可。既定は `docker.io/alpine/git`。**Task をクラスタに再適用**してから PipelineRun を再作成（`oc apply -f tekton/tasks/`） |
| docker.io が禁止のクラスタ | Pipeline / PipelineRun の `git-image`・`maven-image` を社内ミラー URL に変更 |

---

## OpenShift デプロイ（副次）

お客様向けの手順は [openshift/README.md](openshift/README.md) を参照してください。

OpenShift Container Platform に Tomcat 8 アプリを載せるサンプルです。**edge** / **reencrypt** / **passthrough** でマニフェストをディレクトリ分けしています。

## 構成の種類

| パターン | ディレクトリ | TLS 終端 | ルーター → Pod | パス置換 |
|----------|--------------|----------|----------------|----------|
| **edge** | `openshift/edge/` | ルーター | HTTP 8080 | 可 |
| **reencrypt** | `openshift/reencrypt/` | ルーター | HTTPS 8443 | 可 |
| **passthrough** | `openshift/passthrough/` | Tomcat | HTTPS 8443（透過） | 不可 |

**いずれか一方のディレクトリだけ** `oc apply -f` すること。

## ディレクトリ

```
sample-app/
├── ContainerFile
├── tomcat-ssl/
│   ├── conf/
│   └── regenerate-openshift-tls.sh
└── openshift/
    ├── namespace.yaml
    ├── README.md
    ├── edge/
    ├── reencrypt/
    └── passthrough/
```

## デプロイ

```bash
# edge
oc apply -f openshift/namespace.yaml
oc apply -f openshift/edge/

# reencrypt または passthrough
./tomcat-ssl/regenerate-openshift-tls.sh
oc apply -f openshift/namespace.yaml
oc apply -f openshift/reencrypt/
# oc apply -f openshift/passthrough/
```

## 確認

```bash
oc get pods,svc,route -n sample-tomcat
HOST=$(oc get route sample-tomcat -n sample-tomcat -o jsonpath='{.spec.host}')

# edge / reencrypt
curl -sk "https://${HOST}/aaa/"

# passthrough
curl -sk "https://${HOST}/sample/"
```

```bash
# Pod 内
oc exec -n sample-tomcat deploy/sample-tomcat -- \
  curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/sample/    # edge のみ

oc exec -n sample-tomcat deploy/sample-tomcat -- \
  curl -sk -o /dev/null -w "%{http_code}\n" https://127.0.0.1:8443/sample/  # reencrypt / passthrough
```

## 証明書の再生成

`regenerate-openshift-tls.sh` は `reencrypt/` の ConfigMap / Secret / Route を更新し、ConfigMap / Secret を `passthrough/` にコピーします。

## イメージビルド（参考）

```bash
mvn package
podman build -f ContainerFile -t <レジストリ>/sample-tomcat:v1 .
podman push <レジストリ>/sample-tomcat:v1
```
