# 附錄:推薦資源、認證與工具

> 精選的官方文件、書籍、課程與工具清單。學習過程中卡關時,回來這裡找對應資源。

---

## 官方文件(最權威,務必以此為準)

| 主題 | 連結 | 說明 |
|------|------|------|
| Kubernetes | <https://kubernetes.io/docs/> | 官方文件,品質極高,有中文版 (<https://kubernetes.io/zh-cn/docs/>) |
| Amazon EKS | <https://docs.aws.amazon.com/eks/> | EKS 官方文件 |
| EKS Best Practices | <https://aws.github.io/aws-eks-best-practices/> | 生產環境最佳實務,強烈推薦 |
| eBPF | <https://ebpf.io/> | eBPF 官方入口,有教學與文件 |
| Cilium | <https://docs.cilium.io/> | eBPF 在 K8s 的旗艦專案 |
| Docker | <https://docs.docker.com/> | Docker 官方文件 |

---

## 互動式 / 實作平台(動手做)

| 平台 | 連結 | 說明 |
|------|------|------|
| Kubernetes 官方教學 | <https://kubernetes.io/docs/tutorials/> | 內建互動式 minikube 環境 |
| EKS Workshop | <https://www.eksworkshop.com/> | AWS 官方免費實作工作坊,**強烈推薦** |
| Killercoda | <https://killercoda.com/> | 免費的瀏覽器 K8s 沙箱(含 CKA 模擬場景) |
| Play with Kubernetes | <https://labs.play-with-k8s.com/> | 免費線上 K8s 環境 |

---

## 書籍

| 書名 | 作者 | 適合階段 |
|------|------|----------|
| 《Kubernetes Up & Running》 | Brendan Burns 等 | K8s 入門~中階 |
| 《Kubernetes in Action》 | Marko Lukša | K8s 深入(經典) |
| 《Learning eBPF》 | Liz Rice | eBPF 入門首選 |
| 《BPF Performance Tools》 | Brendan Gregg | eBPF / 效能觀測進階(工具大全) |
| 《The Linux Command Line》 | William Shotts | Linux 基礎(免費 PDF) |

---

## 認證(可作為學習目標與履歷加分)

| 認證 | 全名 | 說明 |
|------|------|------|
| **CKA** | Certified Kubernetes Administrator | **最推薦**,純實作考試,考綱就是最好的學習大綱 |
| CKAD | Certified Kubernetes Application Developer | 偏應用開發者 |
| CKS | Certified Kubernetes Security Specialist | K8s 安全,需先有 CKA |
| AWS SAA | Solutions Architect Associate | AWS 基礎,對 EKS 有幫助 |

> 💡 **建議**:把 **CKA** 當作階段 1(Kubernetes)的學習目標。它是 100% 上機實作,逼你真的會用 `kubectl`,而不是死背。考綱本身就是一份完美的學習檢核表。

---

## 必裝工具清單

### 本機 K8s 環境
```bash
kubectl        # K8s 命令列工具(必裝)
kind           # 用 Docker 跑 K8s 叢集(學習推薦)
minikube       # 本機單節點 K8s(另一選擇)
helm           # K8s 套件管理器(Chart)
k9s            # 終端機 K8s 視覺化管理(超好用,強烈推薦)
kubectx/kubens # 快速切換叢集與命名空間
stern          # 同時看多個 Pod 的日誌
```

### EKS / AWS
```bash
aws            # AWS CLI
eksctl         # 建立/管理 EKS 叢集的官方工具
terraform      # 基礎設施即程式碼(進階)
```

### eBPF
```bash
bpftrace       # eBPF 單行追蹤工具
bcc-tools      # eBPF 工具集(execsnoop, opensnoop...)
bpftool        # 檢視與管理 eBPF 程式
```

---

## YouTube / 影音(輔助,別只看不做)

- **TechWorld with Nana** — K8s / DevOps 入門講解清楚
- **CNCF / KubeCon** 官方頻道 — 進階主題與實戰分享
- **Brendan Gregg** 的演講 — eBPF 與效能觀測

---

## 社群

- CNCF Slack(`#kubernetes-users`、`#cilium` 等頻道)
- Reddit `r/kubernetes`
- 台灣社群:CNTUG(Cloud Native Taiwan User Group)

---

> 📌 **提醒**:資源是輔助,**真正的學習發生在你動手敲指令、弄壞東西再修好的時候**。每學一個主題,就回到對應章節做完「動手練習」與「檢核點」。
