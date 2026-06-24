# 第一階段:Kubernetes 核心 (Kubernetes Core)

> 本階段是整個學習計畫的主菜。內容對應官方 Kubernetes 文件 (kubernetes.io) 的觀念架構,但全程以**原創繁體中文**撰寫,用自己的話把「為什麼」講清楚,而不是逐字翻譯規格。

## 這份教材的定位

官方文件 [kubernetes.io](https://kubernetes.io/docs/) 永遠是**最佳、最權威的參考來源**,規格更新也最即時。本教材的角色是**中文導讀 (Chinese reading guide)**:幫你建立直覺、把分散的概念串成一條主線、補上「設計理念」與「實務踩雷」這些官方文件不一定講白的部分。

讀法建議:**先讀本教材建立心智模型 (mental model),再回官方文件查精確的欄位與參數。**

如果你想把這段學習變成一張證照,推薦考 **CKA(Certified Kubernetes Administrator)**。CKA 是純實作考試 (hands-on),考的就是你會不會用 `kubectl` 真的把叢集 (cluster) 操起來,跟本教材「大量 YAML + 指令」的訓練方向完全一致。另外還有 CKAD(偏應用開發者)與 CKS(偏安全),可依興趣延伸。

---

## 學習順序與各主題連結

請依序學習,每個檔案都假設你已經懂前一個檔案的內容:

| 順序 | 主題 | 檔案 | 核心問題 |
|------|------|------|----------|
| 1 | 架構與元件 (Architecture & Components) | [01-architecture.md](./01-architecture.md) | K8s 由哪些東西組成?一個指令下去發生了什麼? |
| 2 | Pod 與工作負載 (Pod & Workloads) | [02-pod-workloads.md](./02-pod-workloads.md) | 我的應用程式怎麼跑起來、怎麼自動維持與升級? |
| 3 | 網路與服務 (Networking & Service) | [03-networking-service.md](./03-networking-service.md) | 一直在變動的 Pod 之間怎麼互相找到對方?外面怎麼進來? |
| 4 | 設定與儲存 (Config & Storage) | [04-config-storage.md](./04-config-storage.md) | 設定檔、密碼、資料怎麼跟容器解耦、怎麼持久化? |
| 5 | 安全與排程 (Security & Scheduling) | [05-security-scheduling.md](./05-security-scheduling.md) | 誰能做什麼?Pod 該被排到哪台機器?資源怎麼控管? |
| 6 | Pod 生命週期與優雅關閉 (Pod Lifecycle & Graceful Shutdown) | [06-pod-lifecycle.md](./06-pod-lifecycle.md) | Pod 從生到死經歷什麼?升級/刪除時怎麼不中斷服務? |
| 7 | 容器安全與加固 (Security Context & Pod Hardening) | [07-security-context.md](./07-security-context.md) | 容器以什麼身分執行?怎麼把它關進最小權限的籠子? |
| 8 | 套件管理、進階除錯與可觀測性 (Helm, Debugging & Observability) | [08-helm-debug-observability.md](./08-helm-debug-observability.md) | 怎麼用 Helm 管理部署?壞掉怎麼查?怎麼看到叢集內部? |

> **章節 1–5 是核心主線**,先讀完並通過檢核點;**章節 6–8 是緊接其後的深化**(生命週期細節、容器安全加固、實務工具),建議在進入 EKS 前一併完成。
>
> 學習節奏建議:每個主題「讀完 → 立刻動手做章末練習 → 對著檢核點自我驗收」,不要囤積到最後才實作。Kubernetes 是動詞,不是名詞。

---

## 本機環境設定 (Local Environment Setup)

學 K8s 最大的門檻是「要有一個叢集 (cluster)」。雲端的託管 K8s(EKS / GKE / AKS)要花錢,所以**本機學習一律用輕量叢集**。兩個主流選擇:

| 工具 | 本質 | 適合場景 | 多節點 (multi-node) |
|------|------|----------|---------------------|
| **kind** (Kubernetes IN Docker) | 把整個 K8s 節點 (node) 跑在 Docker 容器裡 | 跟 CI 整合、需要多節點模擬、輕快 | 原生支援 |
| **minikube** | 在 VM 或容器裡跑一個單節點叢集 | 新手友善、附帶豐富 addon(Ingress、dashboard) | 支援但較重 |

> 本教材的範例兩者皆可。如果你想模擬「多節點排程、汙點容忍」這類主題(第 5 章),建議用 **kind 的多節點設定**。

### 前置工具

```bash
# 1) Docker:kind 與 minikube 的底層都需要容器執行環境 (container runtime)
docker version

# 2) kubectl:跟叢集溝通的命令列工具 (command-line tool),整個學習過程都靠它
#   macOS
brew install kubectl
#   Linux(下載官方 binary)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client     # 確認 client 端版本
```

### 方案 A:kind

```bash
# 安裝 kind
brew install kind                       # macOS
go install sigs.k8s.io/kind@latest      # 或用 Go 安裝

# 建立一個單節點叢集(預設名稱 kind)
kind create cluster --name learn

# kind 會自動把 kubeconfig 寫好,直接就能用 kubectl
kubectl cluster-info --context kind-learn
kubectl get nodes                       # 應該看到一個 control-plane 節點
```

多節點設定(模擬真實叢集,第 5 章排程練習用得到):

```yaml
# kind-multinode.yaml — 一個控制平面 + 兩個工作節點
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane     # 控制平面節點 (control plane node)
  - role: worker            # 工作節點 (worker node)
  - role: worker
```

```bash
kind create cluster --name learn --config kind-multinode.yaml
kubectl get nodes -o wide    # 應看到三個節點
```

### 方案 B:minikube

```bash
# 安裝 minikube
brew install minikube                   # macOS

# 啟動(driver 可用 docker)
minikube start --driver=docker --nodes 1

# 確認
kubectl get nodes
minikube status

# 啟用常用 addon(後面章節會用到)
minikube addons enable ingress          # Ingress 控制器 (Ingress Controller)
minikube addons enable metrics-server   # HPA 需要的指標來源
```

### 清理

```bash
kind delete cluster --name learn        # kind
minikube delete                          # minikube
```

### kubectl 暖身(務必先做)

```bash
# 設定方便的別名與自動補完,之後每天都會省到時間
alias k=kubectl
source <(kubectl completion bash)       # zsh 改成 zsh

# 三個最常用的「探索」指令,養成查 explain 的習慣勝過背 YAML
kubectl explain pod.spec.containers      # 直接從 API 看欄位說明
kubectl api-resources                    # 列出叢集支援的所有資源類型
kubectl get all -A                       # 看看叢集現在有什麼東西在跑
```

---

## 整體檢核點 (Overall Checklist)

完成本階段後,你應該能勾起以下每一項。這也是 CKA 的核心能力地圖:

- [ ] 能畫出 K8s 架構圖,說清楚 API Server、etcd、Scheduler、Controller Manager、kubelet、kube-proxy 各自的職責
- [ ] 能說明一個 `kubectl apply` 從你按下 Enter 到 Pod 跑起來中間發生了什麼
- [ ] 能用 Deployment 部署無狀態應用,並完成滾動更新 (rolling update) 與回滾 (rollback)
- [ ] 能說出 Deployment / StatefulSet / DaemonSet / Job / CronJob 各自的適用場景
- [ ] 能透過 Service 讓 Pod 互相連線,理解 ClusterIP / NodePort / LoadBalancer 的差異
- [ ] 能用 Ingress 把多個服務 (Service) 對外暴露在同一個入口
- [ ] 能說明 CoreDNS 如何讓 `service-name.namespace.svc.cluster.local` 解析得出來
- [ ] 能用 ConfigMap 與 Secret 把設定與密碼跟容器映像 (image) 解耦
- [ ] 能解釋 PV / PVC / StorageClass 的關係並完成一次動態供應 (dynamic provisioning)
- [ ] 能用 RBAC 限制某個 ServiceAccount 只能讀某個 Namespace 的 Pod
- [ ] 能設定 requests / limits,並解釋 QoS 等級與 OOMKilled 的關聯
- [ ] 能設定 liveness / readiness / startup 三種探針 (Probe)
- [ ] 能用 HPA 依 CPU 自動擴縮 (autoscale) Pod 數量
- [ ] 能用 nodeSelector / affinity / taints & tolerations 控制 Pod 落在哪些節點上
- [ ] 能說明 Pod 階段與容器狀態,並講出一個 Pod 被刪除時「優雅關閉」的完整時序(endpoint 移除 → preStop → SIGTERM → 寬限期 → SIGKILL)
- [ ] 能用 securityContext 讓容器以非 root、唯讀根檔案系統、丟棄多餘 Capabilities 執行,並用 Pod Security Admission 在 Namespace 層級強制安全基線
- [ ] 能用 Helm 安裝/升級/回滾一個應用,並用 `kubectl debug` 對沒有 shell 的容器進行除錯

---

> 準備好了就從 [01-architecture.md](./01-architecture.md) 開始。記得:每讀完一章,立刻打開終端機動手做一遍。
