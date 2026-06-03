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
├── pipelines/       # sample-app-ci
├── pipelineruns/    # 手動起動例
├── triggers/        # GitHub PR 用 EventListener / Binding / Template / Route
└── rbac/
    ├── pipeline-sa.yaml              # SA pipeline（clone / maven / scan）
    ├── pipeline-scc-pipelines.yaml   # pipelines-scc → pipeline
    ├── pipelines-sa-build.yaml       # SA pipelines-sa-build（image-build）
    ├── pipeline-scc-buildah-1000.yaml   # SCC 利用権 → pipelines-sa-build
    ├── scc-pipelines-buildah-1000.yaml  # SCC 定義（クラスタ管理者が適用）
    └── triggers-eventlistener.yaml   # EventListener 用 SA / Role
```

### 適用順

```bash
export NS=your-namespace
oc project "$NS"

oc apply -f tekton/rbac/pipeline-sa.yaml
oc apply -f tekton/rbac/pipelines-sa-build.yaml
oc apply -f tekton/rbac/pipeline-scc-pipelines.yaml
oc apply -f tekton/rbac/pipeline-scc-buildah-1000.yaml
oc apply -f tekton/tasks/
oc apply -f tekton/pipelines/

# クラスタ管理者が一度だけ適用（非 root Buildah 用 SCC / ClusterRole）
oc apply -f tekton/rbac/scc-pipelines-buildah-1000.yaml

# 過去に anyuid / privileged を付けていた場合は削除（競合防止）
oc delete rolebinding pipeline-anyuid pipeline-privileged pipeline-tekton-buildah -n "$NS" --ignore-not-found

# PipelineRun の git-url / image を編集してから
oc create -f tekton/pipelineruns/sample-app-ci-run.yaml
```

### GitHub PR トリガー（`feature/*` ブランチ）

`feature/xxxx` ブランチから PR が **opened / synchronize / reopened** されたときに CI を自動起動します。PR の head commit を clone し、イメージタグは `pr-<番号>-<短いSHA>`（例: `quay.io/akhino/sample-tomcat:pr-42-a1b2c3d`）です。

**前提**: Tekton Triggers が有効（OpenShift Pipelines 標準）。`ClusterInterceptor` `github` / `cel` が利用可能であること（`oc get clusterinterceptors`）。

```bash
export NS=your-namespace
oc project "$NS"

# 1. 既存 Task / Pipeline / RBAC を適用済みであること（上記「適用順」参照）

# 2. Webhook 署名検証用 Secret（GitHub Webhook 設定と同じ文字列）
WEBHOOK_SECRET='your-random-secret-string'
oc create secret generic github-webhook-secret \
  --from-literal=secretToken="$WEBHOOK_SECRET" \
  -n "$NS"

# 3. Triggers リソース
oc apply -f tekton/rbac/triggers-eventlistener.yaml
oc apply -f tekton/triggers/

# 4. EventListener Pod が Ready になるまで待つ
oc wait --for=condition=Available deployment/el-sample-app-ci-github-pr -n "$NS" --timeout=120s

# 5. GitHub から到達できる URL を確認
EL_URL="https://$(oc get route el-sample-app-ci-github-pr -n "$NS" -o jsonpath='{.spec.host}')"
echo "$EL_URL"
```

**GitHub リポジトリ設定**（Settings → Webhooks → Add webhook）:

| 項目 | 値 |
|------|-----|
| Payload URL | 上記 `EL_URL` |
| Content type | `application/json` |
| Secret | `WEBHOOK_SECRET` と同じ値 |
| Events | **Pull requests** のみ |

PR 作成後の確認:

```bash
tkn pipelinerun list -n "$NS" -l tekton.dev/trigger=github-pr-feature
tkn pipelinerun logs -f -n "$NS" -l tekton.dev/trigger=github-pr-feature
```

プッシュ先レジストリを変える場合は `tekton/triggers/eventlistener-github-pr.yaml` の `image-registry` パラメータを編集してください。

`image-build` は [Red Hat ドキュメント（非 root Buildah）](https://docs.redhat.com/ja/documentation/red_hat_openshift_pipelines/1.13/html/securing_openshift_pipelines/unprivileged-building-of-container-images-using-buildah) に沿い、**UID 1000（build ユーザー）**・専用 SA `pipelines-sa-build`・SCC `pipelines-scc-buildah-1000` で Tekton 内の buildah を実行します。`capabilities.add` は使わず、`allowPrivilegeEscalation: true` で SETUID/SETGID を有効にします。

Quay へ push する場合（`unauthorized` はほぼ認証未設定）:

```bash
# Quay.io → Account Settings → Robot Account 等でトークン発行
# ロボットの場合: ユーザー名は org+robot 形式（例: akhino+ci-push）
oc create secret docker-registry quay-push-secret \
  --docker-server=quay.io \
  --docker-username='akhino+ROBOT_NAME' \
  --docker-password='YOUR_QUAY_TOKEN' \
  -n "$NS"

# リポジトリ akhino/sample-tomcat への push 権限をロボットに付与すること
oc apply -f tekton/pipelineruns/sample-app-ci-run.yaml  # docker-credentials 有効済み
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
| `uid_map` / capabilities で Pod 拒否 | [RH 非 root Buildah](https://docs.redhat.com/ja/documentation/red_hat_openshift_pipelines/1.13/html/securing_openshift_pipelines/unprivileged-building-of-container-images-using-buildah) どおり `scc-pipelines-buildah-1000` + `pipelines-sa-build` を適用 |
| Quay push `unauthorized` | `quay-push-secret` を作成（`--docker-server=quay.io`）。ロボットに `akhino/sample-tomcat` への write 権限。PipelineRun の `docker-credentials` が有効か確認 |
| `short-name resolution ... cannot prompt without a TTY` | `FROM` を `docker.io/library/...` の完全修飾名に（`ContainerFile` 参照）。Task 再適用後に git 取得からやり直す |
| `fetch-repository` の ImagePullBackOff | `registry.redhat.io` は未認証だと pull 不可。既定は `docker.io/alpine/git`。**Task をクラスタに再適用**してから PipelineRun を再作成（`oc apply -f tekton/tasks/`） |
| docker.io が禁止のクラスタ | Pipeline / PipelineRun の `git-image`・`maven-image` を社内ミラー URL に変更 |
| PR トリガーで PipelineRun が作られない | GitHub Webhook の Recent Deliveries で HTTP 202/200 を確認。`feature/` 以外のブランチは CEL で除外。`oc logs deploy/el-sample-app-ci-github-pr -n "$NS"` |
| Webhook `401` / `403` | `github-webhook-secret` の `secretToken` が GitHub Webhook Secret と一致するか |

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
