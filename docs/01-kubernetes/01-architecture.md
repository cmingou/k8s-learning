# 01 - Kubernetes 架構與元件 (Architecture & Components)

> 目標:理解 Kubernetes 不是一個程式,而是一群會互相協作的元件。讀完你要能回答:「我下了一個 `kubectl apply`,接下來到底發生了什麼?」

---

## 1. 先建立核心直覺:聲明式 (Declarative) 與調和迴圈 (Reconciliation Loop)

學 K8s 之前先記住一句話:**你描述「想要的狀態」,K8s 負責讓現實逼近它。**

傳統運維是「命令式 (imperative)」:你一步步下指令——啟動這個程式、重啟那個服務。Kubernetes 是「聲明式 (declarative)」:你寫一份 YAML 說「我要 3 個這樣的 Pod」,然後 K8s 內部有一堆控制器 (Controller) 不斷做這件事:

```
觀察現況 (observe) → 比對期望 (diff) → 採取行動拉近差距 (act) → 再觀察 ...
```

這個永不停止的迴圈就是**調和迴圈 (reconciliation loop)**,也叫控制迴圈 (control loop)。這是理解 K8s 一切行為的鑰匙。為什麼刪掉一個 Pod 它會自己長回來?因為某個控制器發現「現況 2 個 ≠ 期望 3 個」,於是補了一個。為什麼這個設計好?因為**它天生自我修復 (self-healing)**——你不需要寫「萬一掛了要重啟」的邏輯,差距出現時控制器自然會去補。

---

## 2. 整體架構鳥瞰

一個叢集 (cluster) 分成兩種角色的節點 (node):

- **控制平面 (Control Plane)**:叢集的大腦,做決策。
- **工作節點 (Worker Node)**:真正跑你的容器 (container) 的地方,執行決策。

```mermaid
flowchart TB
    subgraph CP["控制平面 Control Plane"]
        API["kube-apiserver<br/>(唯一入口/前台)"]
        ETCD[("etcd<br/>叢集狀態資料庫")]
        SCHED["kube-scheduler<br/>決定 Pod 排到哪台"]
        CM["kube-controller-manager<br/>各種調和迴圈"]
        CCM["cloud-controller-manager<br/>(對接雲端)"]
        API <--> ETCD
        SCHED --> API
        CM --> API
        CCM --> API
    end

    subgraph W1["Worker Node 1"]
        K1["kubelet"]
        P1["kube-proxy"]
        CR1["容器執行環境<br/>container runtime"]
        POD1["Pods"]
        K1 --> CR1 --> POD1
    end

    subgraph W2["Worker Node 2"]
        K2["kubelet"]
        P2["kube-proxy"]
        CR2["container runtime"]
        POD2["Pods"]
        K2 --> CR2 --> POD2
    end

    USER["你 / kubectl"] --> API
    API <--> K1
    API <--> K2
```

關鍵心法:**所有元件都只跟 API Server 講話,彼此不直接溝通。** API Server 是整個系統的中央交換機,etcd 是它背後唯一的真實資料來源 (source of truth)。這種「星狀、以 API Server 為樞紐」的設計讓系統好擴充、好除錯——你要追任何問題,看 API Server 就對了。

---

## 3. 控制平面元件 (Control Plane Components)

### 3.1 kube-apiserver — 唯一的前門

API Server 是叢集的 **REST API 入口**。所有人(你的 kubectl、其他元件、Pod 內的程式)要讀或改叢集狀態,都得透過它。它負責:

- **認證 (Authentication)**:你是誰?
- **授權 (Authorization)**:你能不能做這件事?(這就是 RBAC,見第 5 章)
- **准入控制 (Admission Control)**:這個請求合不合規?要不要改寫?(例如自動注入預設值)
- **驗證與寫入 etcd**:把資源存進唯一的資料庫。

它是**無狀態 (stateless)** 的——狀態全在 etcd,所以 API Server 可以水平擴充多份做高可用 (HA)。

### 3.2 etcd — 叢集的記憶

etcd 是一個**分散式鍵值資料庫 (distributed key-value store)**,保存叢集裡每一個物件的完整狀態。它用 Raft 共識演算法保證多副本資料一致。

> 為什麼 etcd 這麼重要?因為**整個叢集的真相都在這裡**。etcd 沒了,叢集就失憶了。正式環境一定要定期備份 etcd(`etcdctl snapshot save`),這也是 CKA 必考題。

```bash
# CKA 經典:備份 etcd(需在 control-plane 節點上、給對憑證)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 3.3 kube-scheduler — 決定 Pod 落在哪

當你建立一個 Pod,它一開始是「未排程 (unscheduled)」的——還沒指定要跑在哪台節點。Scheduler 的工作就是幫每個新 Pod **挑一台最合適的節點**,然後把決定寫回 API Server(更新 Pod 的 `nodeName` 欄位)。

它怎麼挑?兩階段:

1. **過濾 (Filtering / Predicates)**:刷掉不符合條件的節點。例如資源不夠、汙點 (taint) 擋住、節點選擇器 (nodeSelector) 不符。
2. **評分 (Scoring / Priorities)**:對剩下的節點打分,選最高分。例如盡量分散、資源最空閒優先。

> 重點:**Scheduler 只負責「做決定」,不負責「真的把容器跑起來」。** 它只是把 Pod 跟節點配對。實際啟動容器是 kubelet 的事。第 5 章會深入排程細節。

### 3.4 kube-controller-manager — 一堆調和迴圈的集合

還記得第 1 節的調和迴圈嗎?Controller Manager 就是一個程序,裡面打包了**幾十個內建控制器**,各自盯著一種資源:

| 控制器 | 它在盯什麼 |
|--------|-----------|
| Deployment / ReplicaSet Controller | Pod 數量有沒有等於期望 |
| Node Controller | 節點有沒有掛掉(失聯就標記並驅逐 Pod) |
| Job Controller | 一次性任務有沒有跑完 |
| Endpoints / EndpointSlice Controller | Service 後面該連到哪些 Pod |
| ServiceAccount Controller | 每個 Namespace 有沒有預設的 ServiceAccount |

每個控制器都在跑「觀察 → 比對 → 行動」。把它們集中在一個程序只是為了部署方便,概念上它們是獨立的。

### 3.5 cloud-controller-manager — 對接雲端(選用)

如果叢集跑在雲上(AWS / GCP / Azure),這個元件負責把 K8s 概念翻譯成雲端資源:建立 LoadBalancer、掛載雲端磁碟、管理節點生命週期。本機學習(kind / minikube)通常沒有它。

---

## 4. 工作節點元件 (Node Components)

### 4.1 kubelet — 節點上的執行者

kubelet 是跑在**每一台節點上**的代理程式 (agent)。它的職責很單純但很核心:

- 向 API Server 領取「我這台該跑哪些 Pod」。
- 透過容器執行環境真的把容器啟動起來。
- 持續回報 Pod 與節點的健康狀態給 API Server。
- 執行探針 (Probe) 檢查容器存活與就緒(見第 5 章)。

> kubelet 是「Scheduler 決定」與「容器實際執行」之間的橋。Scheduler 說「Pod X 給節點 A」,節點 A 上的 kubelet 看到後才動手把它跑起來。

### 4.2 容器執行環境 (Container Runtime)

真正負責拉映像 (image)、啟動/停止容器的底層程式。K8s 透過標準介面 **CRI (Container Runtime Interface)** 跟它溝通,所以可以換不同實作:

- **containerd**(目前最主流)
- **CRI-O**

> 歷史補充:早期用 Docker,後來 K8s 在 1.24 版移除了 Docker 的相容墊片 (dockershim),現在直接用 containerd。映像格式還是相容的,所以對你寫 YAML 沒影響。

### 4.3 kube-proxy — 讓 Service 的虛擬 IP 能通

kube-proxy 也跑在每台節點上,負責實作 **Service 的網路規則**。當你建立一個 Service,它會有一個虛擬 IP(ClusterIP),kube-proxy 在每台節點上設定 iptables / IPVS 規則,讓送到這個虛擬 IP 的流量被導到後面真正的 Pod。第 3 章會詳述。

---

## 5. 完整劇本:一個 `kubectl apply` 發生了什麼?

把上面所有元件串起來。假設你執行:

```bash
kubectl apply -f deployment.yaml   # 內容:要 3 個 nginx Pod
```

```mermaid
sequenceDiagram
    participant U as kubectl
    participant A as API Server
    participant E as etcd
    participant DC as Deployment Controller
    participant RS as ReplicaSet Controller
    participant S as Scheduler
    participant K as kubelet (Node A)
    participant CR as container runtime

    U->>A: 1. 送出 Deployment 物件
    A->>A: 2. 認證/授權/准入/驗證
    A->>E: 3. 寫入 etcd(期望:3 個 Pod)
    A-->>U: 4. 回 200 OK(此時還沒有 Pod!)
    DC->>A: 5. 觀察到新 Deployment,建立 ReplicaSet
    RS->>A: 6. 觀察到 ReplicaSet,建立 3 個 Pod(尚未排程)
    S->>A: 7. 觀察到未排程 Pod,挑節點並寫回 nodeName
    K->>A: 8. Node A 的 kubelet 發現有我的 Pod
    K->>CR: 9. 呼叫 runtime 拉映像、起容器
    K->>A: 10. 回報 Pod 狀態 Running
```

幾個常被誤會的重點:

1. **第 4 步 API Server 就回你 OK 了**,但這時候一個 Pod 都還沒起來。`kubectl apply` 成功只代表「期望狀態被記錄了」,不代表「東西跑起來了」。要看實際狀態得 `kubectl get pods -w`。
2. **沒有任何元件直接命令對方**。每一步都是「某元件觀察 API Server 的變化,然後做出反應」。這就是調和迴圈在多個層級疊加運作。
3. **Deployment → ReplicaSet → Pod** 是三層委派,每層各有控制器。第 2 章會講清楚為什麼要這樣分層。

```bash
# 親眼看這個過程
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods         # 一次看三層:Deployment、ReplicaSet、Pod
kubectl get pods -o wide -w        # -w 持續觀察,看 Pod 從 Pending → Running
kubectl describe pod <pod-name>    # Events 區段是排程與啟動過程的時間軸
```

---

## 6. 物件的共通結構:每個 YAML 都長這樣

K8s 裡所有資源都是「物件 (object)」,都遵循同一套骨架。理解這套骨架,看任何 YAML 都不陌生:

```yaml
apiVersion: apps/v1        # 這個物件屬於哪個 API 群組與版本
kind: Deployment           # 物件類型
metadata:                  # 身分資訊:名字、命名空間、標籤
  name: my-app
  namespace: default
  labels:
    app: my-app
spec:                      # 「期望狀態 (desired state)」——你寫的部分
  replicas: 3
  # ...
status:                    # 「實際狀態 (actual state)」——K8s 自動填,你別手動改
  readyReplicas: 3
```

核心二元對立:**`spec` 是你想要的,`status` 是現在的。** 所有控制器活著就為了讓 `status` 追上 `spec`。

> 善用 `kubectl explain` 自學任何欄位,不用死背:
> ```bash
> kubectl explain deployment.spec.strategy   # 直接從 API 讀欄位文件
> kubectl explain pod --recursive            # 展開整棵欄位樹
> ```

---

## 7. 命名空間 (Namespace) 與標籤 (Label) 先打底

雖然細節在第 5 章,但這兩個概念貫穿全書,先建立印象:

- **Namespace**:叢集內的「邏輯隔離分區」,把資源分組(例如 `dev` / `prod`)。同一個 Namespace 裡名字不能重複,不同 Namespace 可以同名。
- **Label**:貼在物件上的鍵值標籤,例如 `app=nginx`。K8s 大量靠標籤做「選擇 (selecting)」——Service 靠標籤找到要服務的 Pod、Deployment 靠標籤管理它的 Pod。標籤是 K8s 把鬆散物件「綁在一起」的黏著劑。

```bash
kubectl get pods -A                       # -A = 所有命名空間
kubectl get pods -l app=nginx             # 用標籤篩選
kubectl get pods --show-labels            # 看每個 Pod 的標籤
```

---

## 動手練習

1. 用 kind 或 minikube 起一個叢集,執行 `kubectl get nodes -o wide`,辨認出哪個是 control-plane。
2. 執行 `kubectl get pods -n kube-system`,找出 apiserver、etcd、scheduler、controller-manager、coredns、kube-proxy 這些系統 Pod,並用 `kubectl describe` 看其中一個跑在哪台節點。
3. 部署一個簡單的 nginx Deployment(`kubectl create deployment web --image=nginx`),然後用 `kubectl get deploy,rs,pods` 觀察三層結構。
4. 手動刪掉那個 Deployment 底下的一個 Pod(`kubectl delete pod <name>`),再 `kubectl get pods`,觀察它如何自己長回來——這就是調和迴圈。
5. 用 `kubectl describe pod <name>` 看 Events,把它跟第 5 節的劇本對照,找出「Scheduled」與「Started container」兩個事件。

---

## 本章檢核點 (Checklist)

- [ ] 能用自己的話解釋「聲明式」與「調和迴圈」,並舉一個自我修復的例子
- [ ] 能畫出控制平面與工作節點的元件圖,並說明「所有元件只跟 API Server 講話」的意義
- [ ] 能說出 API Server、etcd、Scheduler、Controller Manager 各自的職責
- [ ] 能說出 kubelet、容器執行環境、kube-proxy 各自的職責
- [ ] 能完整講出一個 `kubectl apply` 從送出到 Pod Running 的流程
- [ ] 理解 `kubectl apply` 成功不等於 Pod 已經跑起來,知道要看 `status` 或 `kubectl get pods`
- [ ] 能說出任何 K8s 物件的共通結構(apiVersion / kind / metadata / spec / status),並區分 spec 與 status
- [ ] 知道用 `kubectl explain` 自己查欄位,而不是死背 YAML

> 下一章:[02-pod-workloads.md](./02-pod-workloads.md) — 把應用程式真的跑起來、自動維持、平滑升級。
