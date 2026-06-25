# k8s-learning

從 Linux 基礎一路到 **Kubernetes / EKS / eBPF** 的結構化繁體中文學習筆記。

## 📖 線上閱讀(手機友善)

本筆記已做成可在手機上像網頁瀏覽的文件網站(MkDocs Material,含搜尋、側邊導覽、深色模式):

> **https://cmingou.github.io/k8s-learning/**

> ℹ️ 首次啟用需到 GitHub repo → **Settings → Pages → Build and deployment**,把 **Source** 設為 **GitHub Actions**。之後每次推送到部署分支,[deploy-docs workflow](.github/workflows/deploy-docs.yml) 會自動重建網站。

## 📂 內容結構

所有教材原始檔在 [`docs/`](./docs/) 底下:

| 階段 | 目錄 |
|------|------|
| 0・前置基礎(Linux / 容器 / 網路 / AWS) | [`docs/00-prerequisites/`](./docs/00-prerequisites/) |
| 1・Kubernetes 核心(8 章) | [`docs/01-kubernetes/`](./docs/01-kubernetes/) |
| 2・EKS | [`docs/02-eks/`](./docs/02-eks/) |
| 3・eBPF | [`docs/03-ebpf/`](./docs/03-ebpf/) |
| 附錄・資源 | [`docs/99-resources/`](./docs/99-resources/) |

## 🛠 本機預覽

```bash
pip install -r requirements.txt
mkdocs serve              # 開 http://127.0.0.1:8000 即時預覽
```

教材為原創繁體中文,涵蓋官方文件相同觀念並保留中英對照;官方來源以 [kubernetes.io](https://kubernetes.io/zh-cn/docs/) / [EKS](https://docs.aws.amazon.com/eks/) / [ebpf.io](https://ebpf.io/) 為最終依據。
