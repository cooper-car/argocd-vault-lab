# ArgoCD 實戰教材：用 app-of-apps 部署 Vault + Vault Secrets Operator

> 一份從零開始、可完整跟做的 GitOps 教材。你會在本機 kind cluster 上，用 ArgoCD 以 **app-of-apps** 模式部署 HashiCorp Vault 與 Vault Secrets Operator（VSO），最終讓兩個彼此隔離的 namespace 各自從 Vault 同步出一個 Kubernetes Secret。

---

## 這份教材要教會你什麼

跟完之後，你應該能清楚回答並實作以下幾件事：

1. **ArgoCD 的最小單位 `Application`**：`source` / `destination` / `syncPolicy` 三個問題分別在回答什麼。
2. **同一個 `repoURL` 的兩種語意**：指向 Helm repo（用 chart 版本）vs 指向 Git repo（用 path + branch）。
3. **app-of-apps**：為什麼「一個 App 可以生出一堆 App」，以及它如何把「手動 apply」收斂成「只 push」。
4. **sync waves**：如何用它排出跨元件的部署順序（namespace → Vault → operator → CR）。
5. **VSO 的正確心智模型**：operator 是 cluster 級、裝一次；真正放進各 namespace 的是「叫 operator 去同步」的自訂資源（CR）。
6. **Vault 與 Kubernetes 的信任鏈**：KV secret、Kubernetes auth、policy、role、ServiceAccount 如何一路串到 VSO 的欄位。

---

## 你會用到的工具與版本

| 工具 | 版本 | 說明 |
|---|---|---|
| kind | v0.32.0 | 本機 Kubernetes（node image `kindest/node:v1.36.1`） |
| kubectl / helm | 最新 | 透過 Homebrew 安裝 |
| argocd CLI | 最新 | 透過 Homebrew 安裝 |
| ArgoCD（argo-cd chart） | 10.3.2 | 部署 Argo CD 3.x |
| Vault（vault chart） | 0.34.0 | app 版本 Vault 2.0.3，走 **dev 模式** |
| Vault Secrets Operator（chart） | 1.5.0 | CR schema：`secrets.hashicorp.com/v1beta1` |

> **教學前提**：本機需先安裝並啟動 Docker（Docker Desktop / OrbStack 皆可），kind 靠 Docker container 當 node。

---

## 最終架構全貌

先建立心智圖，後面每一章都在往這棵樹上長節點。

```
[人工只做兩次 kubectl apply：cluster + root-app；其餘全是 git push]

  bootstrap/root-app.yaml   ← 唯一手動 apply 的 Application
        │  （source = apps/ 整個資料夾；持續管理，不是一次性 trigger）
        ▼
     root (Application)
        │
        ├─(wave 0)─► namespaces          source = 本 repo /manifests/namespaces
        │                └─► Namespace: infra / test01 / test02
        │
        ├─(wave 1)─► vault               source = 上游 Helm repo（dev 模式，裝進 infra）
        │
        ├─(wave 2)─► vault-secrets-operator  source = 上游 Helm repo（裝進 infra，帶 CRD）
        │
        ├─(wave 3)─► test01-secrets       source = 本 repo /manifests/test01
        │                └─► ServiceAccount + VaultConnection + VaultAuth + VaultStaticSecret
        │                       └─► K8s Secret: test01-synced（值來自 Vault）
        │
        └─(wave 3)─► test02-secrets       source = 本 repo /manifests/test02
                         └─► ... K8s Secret: test02-synced
```

**兩個關鍵觀察**（整份教材的核心）：

- `vault` / `vault-secrets-operator` 的 source 指向 **外部 Helm repo**；`namespaces` / `test01-secrets` / `test02-secrets` 的 source 指向 **你自己 repo 的路徑**。這就是「同一個欄位、兩種語意」。
- `root → namespaces` 與 `namespaces → 三個 Namespace` 的箭頭語意不同：前者 render 出來的內容「本身是 Application」，後者 render 出來的是「普通 K8s 資源」。app-of-apps 的魔術就在這個差別。

---

## 概念速記（開工前先讀一遍）

**Application**：ArgoCD 的最小單位，宣告「我期望某個來源的內容，被同步到某個叢集的某個位置」。三個核心欄位：

- `source`：期望狀態從哪來。Helm repo → `repoURL` + `chart` + `targetRevision`(chart 版本)；Git repo → `repoURL` + `path` + `targetRevision`(branch/tag/commit)。
- `destination`：部署到哪個 cluster（`server`）、哪個 namespace。
- `syncPolicy`：何時／怎麼同步。`automated.prune`（來源刪了、叢集也刪）、`automated.selfHeal`（有人手改叢集就拉回期望狀態）。

**app-of-apps**：一個 `Application` 的 source 指向「一整包放著其他 Application 的資料夾」。ArgoCD 把那些檔 render 出來、當成自己管理的資源建立起來——於是一個 App 生出一堆 App。好處：新增服務只要往資料夾丟檔 + `git push`，不必再手動 apply。

**sync wave**：在 Application（或資源）上加 annotation `argocd.argoproj.io/sync-wave: "N"`，數字小的先同步、且要等前一波 healthy 才進下一波。用來排跨元件的依賴順序。

**VSO 的部署模型（最容易誤解的地方）**：VSO 是 **cluster 級、一個 cluster 裝一次** 的 operator（帶一組 CRD + 一個 controller）。它會監看整個 cluster。你**不會**在每個 namespace 各裝一次 operator；各 namespace 放的是 CR（`VaultConnection` / `VaultAuth` / `VaultStaticSecret`），operator 讀到後才去 Vault 拿值、產出 K8s Secret。

---

## Chapter 0 — 建立 kind cluster

建立 `kind-argocd-lab.yaml`：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: argocd-lab
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
brew install kind kubectl helm argocd

kind create cluster \
  --config kind-argocd-lab.yaml \
  --image kindest/node:v1.36.1 \
  --wait 120s
```

**✅ 檢查點**

```bash
kubectl get nodes          # 1 control-plane + 2 worker，全部 Ready
kubectl get pods -A        # kube-system 元件 Running
```

> **教學說明**：多 node 只是為了更接近真實排程；單 node 也能跑完整個 lab。`ingress-ready` 標籤與 80/443 對外是預留給 ingress，本 lab 用 port-forward，即使不啟用也無妨。

---

## Chapter 1 — 用 Helm 安裝 ArgoCD

> **為什麼用 Helm 而不是 raw manifest？** 兩者都能裝起 ArgoCD，對後面學到的東西沒有差別。但用 Helm 會產出一份 `values.yaml`，那正是「期望狀態」檔，也是日後讓 ArgoCD 自我管理的素材，跟本 lab 主題一致。
>
> **釐清「三個層」**：(1) 安裝 ArgoCD 本身——bootstrap，一次性、手動；(2) ArgoCD 去管別的 Helm chart——本 lab 主體；(3) ArgoCD 管 ArgoCD——進階。第 1 層用什麼裝，不影響第 2、3 層。

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --version 10.3.2
```

**✅ 檢查點與登入**

```bash
kubectl wait --for=condition=available --timeout=300s -n argocd deployment --all
kubectl get pods -n argocd

# 一個終端機保持開著
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 另一個終端機取得初始 admin 密碼
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

瀏覽器開 `https://localhost:8080`（自簽憑證警告略過），帳號 `admin` + 上面密碼。登入後 Applications 列表是空的，正常。

---

## Chapter 2 — 建立 Git repo 與目錄骨架

> **GitOps 第一守則**：ArgoCD 讀的是 **Git 遠端**，不是你本機硬碟。本機改完檔一定要 `commit + push`，ArgoCD 才看得到。

用 public repo 讓 kind 裡的 ArgoCD 免憑證匿名拉取（private 要額外掛 repo credential，lab 先不折騰）。

```bash
gh repo create argocd-vault-lab --public --clone   # 或到 github.com 手動建再 clone
cd argocd-vault-lab

mkdir -p bootstrap apps manifests/{namespaces,test01,test02}
find . -type d -empty -exec touch {}/.gitkeep \;
echo "# ArgoCD + Vault + VSO lab" > README.md
git add -A && git commit -m "scaffold app-of-apps structure" && git push -u origin main
```

目標結構：

```
argocd-vault-lab/
├── bootstrap/
│   └── root-app.yaml                 # 唯一手動 apply 的東西
├── apps/                             # root 指向這裡；底下每個檔 = 一個子 App
│   ├── namespaces.yaml
│   ├── vault.yaml
│   ├── vault-secrets-operator.yaml
│   ├── test01-secrets.yaml
│   └── test02-secrets.yaml
└── manifests/                        # 真正的 K8s 資源
    ├── namespaces/
    ├── test01/
    └── test02/
```

> 之後把 `<你的帳號>` 全部換成你的 GitHub 帳號。本教材後續以 `cooper-car` 為例。

---

## Chapter 3 — 第一個子 App：namespaces（先走一次完整 GitOps 循環）

先用「手動 apply 這個子 App」跑通，作為理解 root-app 的墊腳石。

`manifests/namespaces/namespaces.yaml`：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: infra
---
apiVersion: v1
kind: Namespace
metadata:
  name: test01
---
apiVersion: v1
kind: Namespace
metadata:
  name: test02
```

`apps/namespaces.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: namespaces
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"          # 最早的一波
spec:
  project: default
  source:
    repoURL: https://github.com/cooper-car/argocd-vault-lab
    path: manifests/namespaces                  # 指 git 路徑（不是 Helm repo）
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    # 不填 namespace：Namespace 是 cluster 級資源
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
git add -A && git commit -m "add namespaces + child app" && git push
kubectl apply -f apps/namespaces.yaml
```

**✅ 檢查點**

```bash
kubectl get application -n argocd     # namespaces → Synced / Healthy
kubectl get ns                        # 出現 infra / test01 / test02
```

> **有感小實驗**：`kubectl delete ns test02`，看 `selfHeal` 幾秒內把它補回來——這就是「宣告式、自我修復」。

> **設計抉擇：為什麼用獨立的 namespaces App，而不是各 App 加 `CreateNamespace=true`？**
> 判準是「這個 namespace 屬於誰」。`infra`/`test01`/`test02` 都被多個 App 共用，交給一個獨立 App 當唯一擁有者，歸屬清楚、能集中掛 label/quota、生命週期完整（從 Git 移除就會被 prune）。若 namespace 只被單一 App 使用，`CreateNamespace=true` 才是更省事的選擇。

---

## Chapter 4 — root-app：app-of-apps 正式成形

`bootstrap/root-app.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/cooper-car/argocd-vault-lab
    path: apps                     # 指向「放子 App 的資料夾」
    targetRevision: main
    directory:
      recurse: false
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
git add -A && git commit -m "add root app-of-apps" && git push
kubectl apply -f bootstrap/root-app.yaml     # 整個 lab 唯一一次 apply Application
```

**✅ 檢查點**

```bash
kubectl get applications -n argocd    # root 與 namespaces 都 Synced/Healthy
```

UI 點開 `root`，會看到 `namespaces` 掛在它底下的樹狀圖。

> **兩個常被擔心、其實沒事的點**：
> 1. 你剛剛手動建的 `namespaces` App，root 會依 `apps/namespaces.yaml` 去「建立」同名 App——因為內容一致，ArgoCD 直接**認領（adopt）**既有那個，不重建、不衝突。從此 `namespaces` 歸 root 管。
> 2. `root-app.yaml` 放在 `bootstrap/`、**不在 `apps/` 底下**，所以 root 不管自己——這是刻意的，它就是整套系統唯一要手動 apply 的東西。

**規則從此改變**：往 `apps/` 丟一個新的 Application YAML → `git push` → root 自動收進來、部署掉。**不再需要 `kubectl apply`。**

---

## Chapter 5 — Vault（dev 模式，裝進 infra）

> **dev 模式**：單 pod、自動 unseal、走 http、root token 已知。lab 最省事，代價是 **pod 一重啟資料就消失**。

`apps/vault.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vault
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"      # 比 namespaces(0) 晚一波
spec:
  project: default
  source:
    repoURL: https://helm.releases.hashicorp.com   # 上游 Helm repo
    chart: vault
    targetRevision: 0.34.0
    helm:
      values: |
        server:
          dev:
            enabled: true
            devRootToken: "root"
        injector:
          enabled: false        # 用 VSO，不需要 agent injector
        ui:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: infra            # 不加 CreateNamespace，交給 namespaces App
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**只 push，不 apply**：

```bash
git add -A && git commit -m "add vault (dev mode) child app" && git push
argocd app get root --refresh          # 不想等預設輪詢就手動刷新
```

**✅ 檢查點**

```bash
kubectl get applications -n argocd     # root / namespaces / vault 都 Synced/Healthy
kubectl get pods -n infra              # vault-0 → Running (1/1)
kubectl exec -n infra vault-0 -- vault status   # Sealed = false

# 看 Vault UI（可選）
kubectl port-forward -n infra svc/vault 8200:8200   # http://localhost:8200，token 填 root
```

---

## Chapter 6 — 手動設定 Vault（整條流程唯一的命令式操作）

> **為什麼這段是手動？** 部署 Vault ≠ 設定 Vault。你得啟用引擎、寫入資料、建立信任關係。這是唯一一段不走 GitOps 的地方（延伸章節會談如何用 Job 收編它）。

進 pod：

```bash
kubectl exec -it -n infra vault-0 -- sh
```

依序執行（dev 模式已自動掛好 `secret/` 這個 KV v2 引擎，不必再 enable）：

```sh
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root

# 1. 寫入兩筆測試 secret（這就是要同步出去的「真值」）
vault kv put secret/test01 username=alice password=p@ss-01
vault kv put secret/test02 username=bob   password=p@ss-02
vault kv get secret/test01

# 2. 啟用 Kubernetes 認證方式
vault auth enable kubernetes

# 3. 設定 auth：叢集內 Vault 用自己的 SA 對 K8s API 做 TokenReview
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443"

# 4. 兩個 policy —— 注意 KV v2 讀取路徑要多一層 data/
vault policy write test01 - <<'EOF'
path "secret/data/test01" {
  capabilities = ["read"]
}
EOF
vault policy write test02 - <<'EOF'
path "secret/data/test02" {
  capabilities = ["read"]
}
EOF

# 5. 兩個 role：綁「哪個 namespace 的哪個 SA」→ 給「哪個 policy」
vault write auth/kubernetes/role/test01 \
    bound_service_account_names=vault-auth \
    bound_service_account_namespaces=test01 \
    policies=test01 ttl=1h
vault write auth/kubernetes/role/test02 \
    bound_service_account_names=vault-auth \
    bound_service_account_namespaces=test02 \
    policies=test02 ttl=1h

exit
```

> 執行 role 時出現 `does not have an audience configured` 的 WARNING 可忽略——audience 非必填，lab 不受影響。

**每一步之後接到 VSO 的哪個欄位**：

| Vault 這側 | 對應到 VSO |
|---|---|
| KV v2 mount = `secret` | `VaultStaticSecret.spec.mount: secret` |
| 資料在 `test01` / `test02` | `VaultStaticSecret.spec.path: test01` |
| auth 路徑 `kubernetes/` | `VaultAuth.spec.method: kubernetes` + `mount: kubernetes` |
| role `test01` / `test02` | `VaultAuth.spec.kubernetes.role: test01` |
| role 綁的 SA `vault-auth` | 你要在 test01/02 建的 `ServiceAccount` 名字 |
| Vault 位址（叢集內） | `VaultConnection.spec.address: http://vault.infra.svc.cluster.local:8200` |

> **重點細節**：role `test01` 只綁 policy `test01`，而 policy `test01` 只允許讀 `secret/data/test01`。所以 test01 的 SA **讀不到** test02——兩個 namespace 的隔離，源頭就在這裡。

---

## Chapter 7 — VSO 與各 namespace 的 CR（收尾）

> VSO chart 的 CRD 放在 `crds/` 目錄，ArgoCD 預設會 `--include-crds` 帶進來，不必額外設定。

### 7-1 VSO 本體（裝進 infra，wave 2）

`apps/vault-secrets-operator.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vault-secrets-operator
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: https://helm.releases.hashicorp.com
    chart: vault-secrets-operator
    targetRevision: 1.5.0
  destination:
    server: https://kubernetes.default.svc
    namespace: infra
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 7-2 各 namespace 的 CR

`manifests/test01/vso.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth                 # 對應 Vault role 綁的 bound_service_account_names
  namespace: test01
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  namespace: test01
spec:
  address: http://vault.infra.svc.cluster.local:8200   # http，dev 模式
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: test01
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: kubernetes                # 對應 enable 的 auth 路徑
  kubernetes:
    role: test01                   # 對應 Vault role
    serviceAccount: vault-auth     # 用這個 SA 的 token 登入 Vault
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: test01
  namespace: test01
spec:
  vaultAuthRef: vault-auth
  mount: secret
  type: kv-v2
  path: test01                     # VSO 自動補成 secret/data/test01
  destination:
    name: test01-synced            # 最後產出的 K8s Secret 名字
    create: true
  refreshAfter: 30s
```

`manifests/test02/vso.yaml`：內容相同，把所有 `test01` 換成 `test02`（namespace、role、path、`destination.name`），`VaultConnection.address` 維持指向 `infra` 不變。

### 7-3 收這兩包 CR 的子 App（wave 3）

`apps/test01-secrets.yaml`（`test02-secrets.yaml` 同理替換名稱與路徑）：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test01-secrets
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: https://github.com/cooper-car/argocd-vault-lab
    path: manifests/test01
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: test01
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 7-4 push，讓它靠 sync wave 一路長到底

```bash
git add -A && git commit -m "add VSO + test01/02 vault secret sync" && git push
argocd app get root --refresh
```

> 因 wave 排序，root 會等 VSO(wave 2) healthy、CRD 就緒後，才建 wave 3 的兩個 secret App。中途若短暫看到 `no matches for kind VaultStaticSecret`，是 CRD 尚未註冊完的暫時狀態，ArgoCD 會自動重試收斂。

---

## Chapter 8 — 驗收（lab 的終點）

```bash
kubectl get applications -n argocd          # 六個 App 全 Synced/Healthy
kubectl get pods -n infra                   # vault-0 + vault-secrets-operator-... Running
kubectl get vaultstaticsecret -A            # test01/test02 已同步

# 關鍵驗收：兩個 namespace 各自冒出從 Vault 同步來的 K8s Secret
kubectl get secret -n test01 test01-synced -o jsonpath='{.data.password}' | base64 -d ; echo
kubectl get secret -n test02 test02-synced -o jsonpath='{.data.password}' | base64 -d ; echo
```

看到 `p@ss-01` 與 `p@ss-02` 分別印出，整條鏈就通了：

```
Vault 裡的真值 ──► VSO 讀取 ──► 在對應 namespace 產出 K8s Secret
（除了 Chapter 6 的一次性設定，全程 GitOps）
```

---

## Chapter 9 — 驗證隔離真的有效（選做）

把 `manifests/test01/vso.yaml` 裡 `VaultStaticSecret.spec.path` 故意改成 `test02`、push。你會看到 test01 這條同步 **失敗、報 403 permission denied**——因為 role `test01` 的 policy 只允許讀 `secret/data/test01`。這親眼證明了 namespace 隔離是由 Vault policy 把關。看完記得改回 `test01`。

---

## 常見坑（Troubleshooting）

| 現象 | 原因 | 解法 |
|---|---|---|
| 改了本機檔，ArgoCD 沒反應 | 忘了 `git push`（ArgoCD 讀 Git 遠端） | commit + push，再 `argocd app get <app> --refresh` |
| policy 明明給了卻讀不到 secret | KV v2 實際路徑多一層 `data/` | policy 寫 `secret/data/test01`，不是 `secret/test01` |
| VSO 認證失敗（403 / permission denied） | role 綁的 SA/namespace 或 policy 不符 | 檢查 `bound_service_account_names/namespaces` 與 `policies` |
| test-secrets 首次 sync 報 `no matches for kind ...` | VSO CRD 尚未就緒 | sync wave 會處理；稍待自動收斂 |
| 重開機後 VSO 全部認證失敗 | dev 模式 Vault 重啟、記憶體資料清空 | 重跑 Chapter 6（K8s 側 CR 不動） |
| namespace 被 prune 後 App 卡住 | 明確 Namespace 物件被移除觸發連鎖刪除 | 確認 Namespace 檔仍在 Git，或調整擁有者 |

---

## 延伸方向

1. **把 Vault 設定也收進 GitOps**：用一個 init `Job`（或 Vault 的 config-as-code）取代 Chapter 6 的手動 `kubectl exec`，讓「設定」也宣告式、可重跑。
2. **用 ApplicationSet 消除重複**：test01/test02 這種同構結構，可用 `ApplicationSet` 的 list/git generator 自動生成，新增 test03 只要加一筆清單。
3. **非 dev 的 Vault**：改用 standalone/HA + 持久化儲存，練 init/unseal 流程與 auto-unseal。
4. **ArgoCD 管 ArgoCD**：把 Chapter 1 的 `values.yaml` 放進 repo，建一個指向 argo-cd chart 的 App，讓 ArgoCD 自我管理（GitOps all the way down）。

---

## 附錄 A — 最終 repo 結構

```
argocd-vault-lab/
├── bootstrap/
│   └── root-app.yaml
├── apps/
│   ├── namespaces.yaml               # wave 0
│   ├── vault.yaml                    # wave 1
│   ├── vault-secrets-operator.yaml   # wave 2
│   ├── test01-secrets.yaml           # wave 3
│   └── test02-secrets.yaml           # wave 3
└── manifests/
    ├── namespaces/namespaces.yaml    # infra / test01 / test02
    ├── test01/vso.yaml               # SA + VaultConnection + VaultAuth + VaultStaticSecret
    └── test02/vso.yaml
```

## 附錄 B — 部署順序與人工介入點

| 順序 | 動作 | 人工介入？ |
|---|---|---|
| 1 | `kind create cluster` | ✅ 手動 |
| 2 | `helm install argocd`（Chapter 1） | ✅ 手動（bootstrap） |
| 3 | `kubectl apply -f bootstrap/root-app.yaml` | ✅ 手動（唯一的 Application apply） |
| 4 | namespaces / vault / vso / test01/02 | ❌ 只 `git push`，root 自動收斂 |
| 5 | Vault KV/auth/policy/role（Chapter 6） | ✅ 手動（唯一的命令式操作） |

> 除了 cluster 建立、ArgoCD bootstrap、root-app apply 這三個 bootstrap 動作，以及 Vault 一次性設定之外，**所有應用的部署都是 `git push` 驅動**。這就是 app-of-apps 想達到的狀態。

---

## 附錄 C — 清理

```bash
kind delete cluster --name argocd-lab
```

---

*本教材的軟體版本以撰寫時（2026-08）的最新穩定版為準；實作時建議用 `helm search repo <chart> --versions` 確認當下版本，並據以調整各 `targetRevision`。*
