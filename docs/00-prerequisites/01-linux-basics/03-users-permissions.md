# Linux 基礎 3:使用者、群組與權限

> 目標:看懂並能修改 Linux 的權限模型。容器安全 (Security Context)、K8s 的 RBAC、檔案掛載權限,全都建立在這套觀念上。

---

## 1. 使用者 (User) 與群組 (Group)

Linux 是多使用者系統。每個使用者有唯一的 **UID**(User ID),每個群組有唯一的 **GID**(Group ID)。

| 類型 | UID | 說明 |
|------|-----|------|
| **root(超級使用者)** | `0` | 最高權限,什麼都能做。`#` 提示字元。 |
| 系統使用者 | 1–999 | 給服務用的(例如 nginx、postgres),通常不能登入。 |
| 一般使用者 | 1000+ | 真人帳號。 |

```bash
whoami           # 我是誰
id               # 我的 UID、GID、所屬群組
id nginx         # 看某個使用者的資訊
cat /etc/passwd  # 所有使用者(每行一個)
cat /etc/group   # 所有群組
groups           # 我屬於哪些群組
```

`/etc/passwd` 每行格式(用 `:` 分隔):

```
使用者名稱:x:UID:GID:說明:家目錄:登入用的 shell
youruser:x:1000:1000:Your Name:/home/youruser:/bin/bash
```

> 🔑 **與容器的連結**:容器裡的行程也以某個 UID 執行。預設常常是 **root (UID 0)**,這是個安全隱憂。K8s 的 `securityContext.runAsUser` 就是用來指定「容器內行程以哪個 UID 跑」,避免用 root。記住這個 UID 觀念。

---

## 2. 檔案權限:rwx 模型

每個檔案/目錄都有三組權限,分別給三種對象:

```
-rwxr-xr--   1  owner  group  ...  file.sh
│└┬┘└┬┘└┬┘
│ │  │  └── 其他人 (others):r-- → 只能讀
│ │  └───── 群組 (group):r-x → 讀 + 執行
│ └──────── 擁有者 (owner/user):rwx → 讀 + 寫 + 執行
└────────── 檔案類型:- 一般檔, d 目錄, l 連結
```

| 權限 | 對檔案 | 對目錄 |
|------|--------|--------|
| `r` (read, 4) | 可讀內容 | 可列出目錄內容 (`ls`) |
| `w` (write, 2) | 可修改 | 可在裡面新增/刪除檔案 |
| `x` (execute, 1) | 可執行 | 可進入該目錄 (`cd`) |

### 數字表示法(八進位)

每組權限 = `r(4) + w(2) + x(1)` 相加:

| 數字 | 權限 | 意義 |
|------|------|------|
| 7 | rwx | 讀寫執行 |
| 6 | rw- | 讀寫 |
| 5 | r-x | 讀 + 執行 |
| 4 | r-- | 唯讀 |
| 0 | --- | 無 |

所以 `chmod 755 file` = 擁有者 `rwx`、群組 `r-x`、其他人 `r-x`。
`chmod 644 file` = 擁有者 `rw-`、其他人 `r--`(一般檔案常見)。

---

## 3. 修改權限與擁有者

```bash
# chmod:改權限
chmod 755 script.sh          # 用數字
chmod +x script.sh           # 加上「執行」權限(常用於讓腳本可執行)
chmod -x script.sh           # 移除執行權限
chmod u+w,g-w,o-r file       # 細部:擁有者加寫、群組去寫、其他人去讀
chmod -R 755 mydir/          # -R 遞迴整個目錄

# chown:改擁有者 / 群組(通常需 sudo)
sudo chown alice file.txt          # 改擁有者為 alice
sudo chown alice:devs file.txt     # 同時改擁有者與群組
sudo chown -R alice:devs mydir/    # 遞迴
```

> 💡 **超常見錯誤**:寫好一個 `script.sh` 卻 `Permission denied` —— 因為忘了 `chmod +x`。

---

## 4. sudo:暫時借用 root 權限

`sudo`(superuser do)讓被授權的使用者以 root(或他人)身分執行單一指令。

```bash
sudo apt update              # 以 root 執行這一個指令
sudo -i                      # 切換成 root 的互動 shell(小心使用)
sudo -u postgres psql        # 以 postgres 使用者身分執行
```

- 比起直接登入 root,`sudo` 更安全:有稽核紀錄、權限可精細控管(`/etc/sudoers`)、不必共享 root 密碼。
- **最小權限原則 (Principle of Least Privilege)**:只給剛好夠用的權限。這個原則貫穿整個雲原生安全(K8s RBAC、AWS IAM 都是同一哲學)。

---

## 5. 特殊權限(進階,但容器會遇到)

| 權限 | 符號 | 作用 |
|------|------|------|
| **SUID** | `s`(在 user 的 x 位) | 執行時以「檔案擁有者」身分跑(例如 `passwd` 指令) |
| **SGID** | `s`(在 group 的 x 位) | 執行時以群組身分跑;目錄則讓新檔繼承群組 |
| **Sticky Bit** | `t`(在 others 的 x 位) | 用於 `/tmp`:只有擁有者能刪自己的檔 |

```bash
ls -l /usr/bin/passwd     # 會看到 -rwsr-xr-x,那個 s 就是 SUID
ls -ld /tmp               # 會看到 drwxrwxrwt,最後的 t 就是 sticky bit
```

> 🔑 **與容器安全的連結**:容器逃逸 (container escape) 攻擊常跟特殊權限、Linux 能力 (Capabilities) 有關。K8s 的 `securityContext` 可以丟棄不必要的 Linux 能力 (`capabilities.drop`)、禁止權限提升 (`allowPrivilegeEscalation: false`)。這些都是這套權限模型的延伸。

---

## 6. Linux 能力 (Capabilities)(觀念預告)

傳統上權限是「全有(root)或全無」。**Linux 能力**把 root 的超能力**拆成幾十個細項**,例如:

| 能力 | 作用 |
|------|------|
| `CAP_NET_ADMIN` | 設定網路介面、iptables |
| `CAP_NET_BIND_SERVICE` | 綁定 1024 以下的特權連接埠 |
| `CAP_SYS_ADMIN` | 一大堆系統管理操作(很危險) |
| `CAP_BPF` | 載入 eBPF 程式(eBPF 章節會用到!) |

```bash
# 查看某行程擁有的能力(需安裝 libcap2-bin)
getpcaps <PID>
capsh --print
```

> 這讓你可以給容器「剛好夠用」的能力,而不是整個 root。eBPF 程式需要 `CAP_BPF`,這正是第 4 章與 eBPF 章節的連結點。

---

## 動手練習

1. **看自己的身分**:執行 `id`、`whoami`、`groups`,理解你的 UID/GID 與所屬群組。打開 `/etc/passwd` 找到自己那行,解讀每個欄位。
2. **權限實作**:
   ```bash
   echo 'echo hello' > script.sh
   ls -l script.sh          # 觀察預設權限
   ./script.sh              # 應該會 Permission denied
   chmod +x script.sh       # 加上執行權限
   ./script.sh              # 現在可以跑了
   ```
3. **數字權限**:把 `script.sh` 改成 `chmod 640`,用 `ls -l` 確認變成 `-rw-r-----`,並說出每組代表什麼。
4. **擁有者**:`sudo chown root script.sh`,觀察你現在還能不能改它的內容。
5. **找特殊權限**:`ls -l /usr/bin/passwd` 找出 SUID 的 `s`;`ls -ld /tmp` 找出 sticky bit 的 `t`。
6. **最小權限思考**:想一想,為什麼讓容器以 root 跑是危險的?(之後 K8s securityContext 會回答你。)

---

## 本節檢核點

- [ ] 理解 UID/GID、root (UID 0) 與一般使用者的差別。
- [ ] 能解讀 `ls -l` 的 `-rwxr-xr--` 權限字串。
- [ ] 會用數字 (`755`/`644`) 與符號 (`+x`, `u+w`) 兩種方式 `chmod`。
- [ ] 會用 `chown` 改擁有者與群組。
- [ ] 理解 `sudo` 與最小權限原則,並知道它對應到 K8s RBAC / AWS IAM 的哲學。
- [ ] 對 Linux 能力 (Capabilities) 有初步概念,知道 `CAP_BPF` 與 eBPF 有關。
- [ ] **理解「容器內行程的 UID」與 K8s `securityContext` 的關係。**

➡️ 下一節(最重要的一節):[命名空間與控制群組 —— 容器的真正原理](./04-namespaces-cgroups.md)
