# sample-app-with-tekton
test
OpenShift 上で **Tekton** が CI（ソース取得 → コードビルド → UT → 静的スキャン → イメージビルド → イメージスキャン）を順に実行するデモ用リポジトリです。アプリは Maven で `target/sample.war` を生成し、`ContainerFile` で Tomcat イメージに載せます。

## 前提

- `oc login` 済み、対象 namespace を選択
- OpenShift Pipelines（Tekton）がインストール済み
- イメージ push 先と、必要なら pull/push 用 Secret

## Tekton リソースの適用

```bash
export NS=your-namespace
oc project "$NS"

oc apply -f tekton/rbac/pipeline-sa.yaml
oc apply -f tekton/rbac/pipelines-sa-build.yaml
oc apply -f tekton/rbac/pipeline-scc-pipelines.yaml
oc apply -f tekton/rbac/pipeline-scc-buildah-1000.yaml
oc apply -f tekton/tasks/
oc apply -f tekton/pipelines/

# クラスタ管理者が一度だけ適用（非 root Buildah 用 SCC）
oc apply -f tekton/rbac/scc-pipelines-buildah-1000.yaml

# 過去に anyuid / privileged を付けていた場合は削除
oc delete rolebinding pipeline-anyuid pipeline-privileged pipeline-tekton-buildah -n "$NS" --ignore-not-found

# git-url / image を編集してから
oc create -f tekton/pipelineruns/sample-app-ci-run.yaml
```

`tekton/` 配下: `tasks/`（各 CI ステップ）、`pipelines/`（`sample-app-ci`）、`pipelineruns/`（手動起動例）、`triggers/`（GitHub PR）、`rbac/`。

## CI の見方

```bash
tkn pipelinerun list -n "$NS"
tkn pipelinerun logs -f -n "$NS" -l app.kubernetes.io/part-of=sample-app-ci

oc get pipelinerun,taskrun -n "$NS"
oc describe pipelinerun <name> -n "$NS"
tkn taskrun list -n "$NS"
```

成功条件: TaskRun が **fetch-repository → code-build → unit-test → static-scan → image-build → image-scan** の順ですべて `Succeeded`（静的・イメージスキャンは dummy）。

## GitHub PR トリガー（任意）

`feature/*` ブランチの PR（opened / synchronize / reopened）で CI を自動起動。イメージタグは `pr-<番号>-<短いSHA>`。Pipeline 完了後、GitHub PR の **Checks** に `tekton/sample-app-ci` として pending / success / failure が表示される（要 GitHub Token）。

```bash
WEBHOOK_SECRET='your-random-secret-string'
oc create secret generic github-webhook-secret \
  --from-literal=secretToken="$WEBHOOK_SECRET" -n "$NS"

# PR Checks 用（repo:status 権限の Classic PAT または fine-grained token）
GITHUB_TOKEN='ghp_xxxxxxxx'
oc create secret generic github-api-token \
  --from-literal=token="$GITHUB_TOKEN" -n "$NS"

# ClusterRoleBinding の namespace を $NS に合わせて適用（未適用だと Route が 503）
sed "s/namespace: sample-tomcat/namespace: ${NS}/" tekton/rbac/triggers-eventlistener.yaml | oc apply -f -
oc apply -f tekton/triggers/
oc wait --for=condition=Available deployment/el-sample-app-ci-github-pr -n "$NS" --timeout=120s
oc get endpoints el-sample-app-ci-github-pr -n "$NS"

EL_URL="https://$(oc get route el-sample-app-ci-github-pr -n "$NS" -o jsonpath='{.spec.host}')"
echo "$EL_URL"
# curl で 503 ではなく 400 が返れば EL Pod は Ready（GitHub は POST で送る）
```

GitHub Webhook: Payload URL = `EL_URL`、Content type = `application/json`、Secret = `WEBHOOK_SECRET`、Events = **Pull requests** のみ（`push` だけでは CI は起動しない）。PR 作成時に 503 だった場合は **Recent Deliveries → Redeliver**。

```bash
tkn pipelinerun list -n "$NS" -l tekton.dev/trigger=github-pr-feature
tkn pipelinerun logs -f -n "$NS" -l tekton.dev/trigger=github-pr-feature
```

レジストリ変更は `tekton/triggers/eventlistener-github-pr.yaml` の `image-registry` を編集。

OpenShift コンソールから PipelineRun へリンクしたい場合は `tekton/triggers/triggertemplate-sample-app-ci-pr.yaml` の `openshift-console-url` 既定値をクラスタのコンソール URL（例: `https://console-openshift-console.apps.cluster.example.com`）に変更する。未設定でも Checks の成否表示は動作する（リンクなし）。

Branch protection でマージを CI 成功時のみにする場合: GitHub → Settings → Branches → Required status checks に **`tekton/sample-app-ci`** を追加。

Quay push 用 Secret の例:

```bash
oc create secret docker-registry quay-push-secret \
  --docker-server=quay.io \
  --docker-username='ORG+ROBOT_NAME' \
  --docker-password='YOUR_QUAY_TOKEN' \
  -n "$NS"
```

## よくある失敗

| 現象 | 対処 |
|------|------|
| UT 失敗 | `tkn taskrun logs <unit-test-run> -n "$NS"` |
| Maven 依存の取得失敗 | クラスタから Maven Central へ出られるか確認 |
| `uid_map` / capabilities で Pod 拒否 | `scc-pipelines-buildah-1000` + `pipelines-sa-build` を適用（[RH 非 root Buildah](https://docs.redhat.com/ja/documentation/red_hat_openshift_pipelines/1.13/html/securing_openshift_pipelines/unprivileged-building-of-container-images-using-buildah)） |
| Quay push `unauthorized` | `quay-push-secret` と PipelineRun の `docker-credentials` |
| `short-name resolution ... cannot prompt without a TTY` | `ContainerFile` の `FROM` を完全修飾名に |
| `fetch-repository` の ImagePullBackOff | `oc apply -f tekton/tasks/` 後に PipelineRun を再作成 |
| Route / Webhook が **503** | `oc get endpoints el-sample-app-ci-github-pr -n "$NS"` が空 → `triggers-eventlistener` SA と `tekton-triggers-eventlistener-clusterroles` のバインドを適用し Pod を再起動 |
| PR トリガーで Run が作られない | Webhook の HTTP 202/200、head ブランチが **`feature/`** 始まり（`feature-foo` は不可）、EventListener のログ |
| Webhook `401` / `403` | `github-webhook-secret` と GitHub Secret の一致 |
| PR に Checks が出ない | `github-api-token` Secret、`repo:status` 権限、`github-status-*` TaskRun のログ |
| Checks が `failure` のまま | `tkn taskrun logs -n "$NS" -l tekton.dev/pipelineTask=github-status-failure` |
