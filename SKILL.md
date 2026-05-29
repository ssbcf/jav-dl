---
name: jav-dl
description: JAV 番号查询与下载工具。支持两种模式：① 查询片名/女优/封面信息 ② BT 下载（自动搜索+筛选+下载+清理）。下载目录通过 $JAV_DL_DIR 环境变量配置，默认为 $HOME/Downloads/av。TRIGGER when: 用户说"下载 XXX-123"、"/jav-dl XXX-123"、"查询 XXX-123"、"XXX-123的片名/女优是什么"、"告诉我XXX-XXX"；用户提到番号 + 需要下载/查询。
---

# JAV 下载技能

## 触发方式（两种模式）

**模式 1：查询信息** — 用户说"XXX-123的片名/女优是什么"、"查询 XXX-123"、"告诉我 XXX-123"
→ 仅搜索片名/女优/封面信息，不下载

**模式 2：下载** — 用户说"下载 XXX-123"或"/jav-dl XXX-123"
→ 完整走搜索→筛选→下载→清理流程

---

## 核心原则

### 一、下载方式优先级

```
.torrent 文件  >>  magnet 磁力链接  >>  流媒体直链（yt-dlp）
   秒连                1~20分钟卡元数据          需 JS 解析/反爬
```

### 二、版本选择优先级

筛选顺序：**无码 > 中文 > 种子数**

```
第一层：  无码(UC/U/破解)  >>  有码(C/原版)
第二层：    中文版本（中文字幕/中文标题）  >  非中文
第三层：      种子数多    >    种子数少
```

| 标签 | 含义 | 优先级 |
|------|------|--------|
| `UC` | 无码破解 + 中文字幕 | 最优 |
| `U` / `无码` / `破解` / `Reducing Mosaic` | 无码（含中文或无中文） | 优先 |
| `C` / `中文字幕` | 有码 + 中文字幕 | 次选 |
| `4K` / `FHD` / `HD` | 分辨率标注 | 同优先级内选画质好的 |
| 无标签 | 有码原版 | 最后 |

### 综合决策树
```
1. 优先选出无码版本（UC > U > 破解 / 无码 / Reducing Mosaic）
2. 无码版本中优先选中文（UC 最优，再 U+中文字幕）
3. 无无码版本时降级到 C / 中文字幕（有码+中文）
4. 确定版本后挑 .torrent 可用的（Nyaa 优先）
5. 同条件挑种子数最多 + 文件最小的
6. 无 .torrent 时用 magnet 兜底
```

---

## 配置

### 下载目录

通过 `$JAV_DL_DIR` 环境变量自定义下载目录，默认为 `$HOME/Downloads/av`：

```bash
# Linux / macOS
export JAV_DL_DIR="$HOME/Downloads/av"

# Windows (Git Bash / WSL)
export JAV_DL_DIR="/c/Users/xxx/Downloads/av"
```

在命令中使用方式：
```bash
# 临时覆盖
JAV_DL_DIR="/mnt/d/Downloads/av" /jav-dl XXX-123

# 或写入 shell 配置文件
echo 'export JAV_DL_DIR="$HOME/Downloads/av"' >> ~/.bashrc
```

### 系统依赖

| 工具 | 用途 | 安装方式 |
|------|------|----------|
| `curl` | HTTP 请求 | 系统自带 / `apt install curl` / `winget install curl` |
| `aria2c` | BT 下载 | `apt install aria2` / `brew install aria2` / `winget install aria2` |

> **Windows 用户**：命令中的 `taskkill /f /im aria2c.exe` 为 Windows 特有。Linux/macOS 下请替换为 `pkill aria2c`。

---

## 模式 1：查询信息（快速流程）

```
仅需 3 步，不涉及下载：

1. Nyaa 搜索 GET /?q={番号}&s=seeders&o=desc → 解析标题获取日文片名 + 女优名
2. 磁力猫搜索 GET /search?word={base64(番号)} → 解析中文标题 + 标签（U/C/UC）
3. javmost.ws 搜索 → 拿封面图 + 更多女优/片商信息（可选）

输出格式：番号 | 片名 | 女优 | 发售日 | 版本标签 | 可用种子数
```

## 模式 2：下载（完整流程）

### 阶段 1：代理检测
检测本地代理（Clash 默认 7890 端口）：
```
curl -s --max-time 3 http://127.0.0.1:7890 >/dev/null && echo "有代理" || echo "无代理"
```
- 有代理 → Nyaa 等境外站走 `-x http://127.0.0.1:7890`
- 无代理 → 仅用磁力猫（直连）
- **BT 数据始终不经过代理**

### 阶段 2：并行搜索（Nyaa + 磁力猫同时发起）

两个来源互不依赖，一次并行请求：

```
# 并行请求 1：Nyaa（.torrent 来源）
curl -s -L --insecure -x http://127.0.0.1:7890 \
  "https://sukebei.nyaa.si/?q={番号}&s=seeders&o=desc"

# 并行请求 2：磁力猫（magnet 来源 + 中文标签）
curl -s -L -c /tmp/clmcookie.txt "https://clm.cc/" -o /dev/null  # 拿 cookie
SEARCH=$(echo -n "{番号}" | base64)
curl -s -L -b /tmp/clmcookie.txt "https://clm59.top/search?word=${SEARCH}&sort=time"
```

解析字段：

| 来源 | 提取内容 |
|------|----------|
| Nyaa | 标题（日文片名+女优）、大小、种子数、日期、`/download/{id}.torrent` 链接、magnet 链接 |
| 磁力猫 | 标题（中文标签）、大小、热度、日期、`/information/{hash}` 详情链接 |

### 阶段 3：结果汇总与筛选

1. **去重合并**：按 info_hash 去重，两个来源互补字段
2. **按决策树排序**：无码优先 → 中文优先 → 种子数降序
3. **展示列表**：`排名 | 大小 | 种子数 | 来源 | 版本标签 | 标题`

**选择逻辑**：列表第一项即为推荐，用户可直接确认或手动指定。

### 阶段 4：获取下载链接并启动

#### 4.1 首选：Nyaa `.torrent` 文件

```
# 下载 .torrent 文件（几 KB，秒下）
curl -s -L -x http://127.0.0.1:7890 --insecure \
  "https://sukebei.nyaa.si/download/{id}.torrent" \
  -o "$JAV_DL_DIR/{番号}/{番号}.torrent"

# 启动 aria2c（元数据内嵌，秒连对等端）
aria2c --file-allocation=none --bt-max-peers=300 --seed-time=0 \
  --console-log-level=notice --dht-entry-point=dht.transmissionbt.com:6881 \
  --bt-tracker="udp://tracker.opentrackr.org:1337/announce,udp://exodus.desync.com:6969/announce,udp://tracker.torrent.eu.org:451/announce,udp://open.stealth.si:80/announce,udp://tracker.moeking.me:6969/announce,udp://tracker.cyberia.is:6969/announce,udp://ipv4.tracker.harry.lu:80/announce,http://sukebei.tracker.wf:8888/announce" \
  "$JAV_DL_DIR/{番号}/{番号}.torrent" \
  -d "$JAV_DL_DIR/{番号}"
```

#### 4.2 备选：磁力猫 magnet（Nyaa 无结果时）

```
1. 进详情页 GET /information/{hash} → 提取 magnet:?xt=urn:btih:{info_hash}
2. 用 info_hash 反查 Nyaa：GET /?q={info_hash} → 若命中，下载 .torrent（切回 4.1 流程）
3. 若 Nyaa 无对应种 → 用 magnet 启动 aria2c（元数据需 DHT，可能慢）
```

#### 4.3 流媒体站（仅用于查片名/封面，不下载）
```
javmost.ws → JS 重度混淆，yt-dlp 无 extractor，仅作信息参考
```

### 阶段 5：监控

aria2c 以 `run_in_background` 启动后：
- 30 秒后读取 `.output` 文件确认元数据已获取、已有下载速度
- 若 2 分钟内 DL:0B → 停止，换种子数更高的版本重试
- 每 15 分钟读取 `.output` 末尾 5 行汇报：`大小 | % | 速率 | ETA`
- 进度 ≥99% 持续 2 分钟无进展 → 任务完成，进入阶段 6

### 阶段 6：下载完成清理

下载完成后自动执行：
```
1. 删除 .torrent 文件        # 种子已不需要
2. 删除 .aria2 控制文件      # aria2c 正常退出时会自动删，否则手动清理
3. 删除 manko.fun.*          # 广告垃圾文件
4. taskkill /f /im aria2c.exe # 确保进程退出
5. 删除空目录（如有）
6. ls -lh 确认最终文件列表
```

---

## 统一重试与容错策略

| 场景 | 策略 |
|------|------|
| Nyaa SSL EOF | `--insecure` + 最多重试 3 次，间隔 2 秒 |
| Nyaa 429/IP限速 | 等待 5 秒后重试，最多 3 次 |
| 磁力猫返回 500 | 检查是否忘记 Base64 编码 → 重试 |
| 磁力猫无 magnet | 跳过该条目，换下一个 |
| 磁力猫域名变化 | 始终 `-L` 跟随重定向，cookie 跨域名复用 |
| magnet 元数据卡住（>2分钟） | 杀掉进程，用 info_hash 去 Nyaa 找 .torrent |
| .torrent 下载 2 分钟无速度 | 杀掉进程，换种子数更多的版本 |
| aria2c 僵尸进程 | 每次启动前 `taskkill /f /im aria2c.exe` |

---

## aria2c 完整启动模板

```
taskkill /f /im aria2c.exe 2>/dev/null
mkdir -p "$JAV_DL_DIR/{番号}"

aria2c --file-allocation=none --bt-max-peers=300 --seed-time=0 \
  --console-log-level=notice --dht-entry-point=dht.transmissionbt.com:6881 \
  --bt-tracker="udp://tracker.opentrackr.org:1337/announce,udp://exodus.desync.com:6969/announce,udp://tracker.torrent.eu.org:451/announce,udp://open.stealth.si:80/announce,udp://tracker.moeking.me:6969/announce,udp://tracker.cyberia.is:6969/announce,udp://ipv4.tracker.harry.lu:80/announce,http://sukebei.tracker.wf:8888/announce" \
  "<.torrent 文件路径或 magnet 链接>" \
  -d "$JAV_DL_DIR/{番号}"
```

**参数说明**：

| 参数 | 作用 |
|------|------|
| `--file-allocation=none` | Windows 下跳过慢速磁盘预分配 |
| `--seed-time=0` | 下载完成即停止，不做种 |
| `--bt-max-peers=300` | 最多连接 300 个对等端 |
| `--console-log-level=notice` | 减少输出噪音，只留关键进度 |
| `--dht-entry-point` | DHT 入口节点，加速首次连接 |
| `--bt-tracker` | 附加 tracker 列表，增加节点发现 |

---

## 容易出问题的点

### 下载元数据（最关键的坑）
1. **magnet 元数据卡住**（最痛点）：磁力链接通过 DHT 获取元数据，可能 1~20 分钟。优先用 Nyaa `.torrent` 文件
2. **.torrent 秒连**：已验证 `magnet 2分钟0B` vs `.torrent 5秒连10节点1MiB/s`
3. **magnet → Nyaa 反查**：从磁力猫拿到 info_hash 后去 Nyaa 搜 `/download/{id}.torrent`，变 magnet 为 .torrent

### Nyaa 相关
1. SSL 不稳定 → 加 `--insecure`，重试 3 次间隔 2 秒
2. IP 限速 → 间隔 1 秒，429 后等 5 秒
3. 种子数不实时 → 标注 27 种实际可能 3-5 在线
4. 走代理 → Clash 节点质量影响连接

### 磁力猫相关
1. Cookie 必须先拿 → 首页获取 PHPSESSID
2. 搜索词必须 Base64 → 明文返回 500
3. `<em>` 标签包裹匹配词 → 解析时 strip HTML
4. 域名常变 → 始终 `-L` 跟随重定向

### BT 下载相关
1. 速度波动极大 → 国内 3 KiB/s ~ 2 MiB/s 都正常
2. 不要 `--all-proxy` → BT 数据走代理连不上
3. 上传量 > 下载量 → 正常，BT 协议边下边传
4. 速度慢换版本 → 种子数/热度更高的优先

### 通用
1. 下载目录 → 确保 `$JAV_DL_DIR` 目录存在（默认为 `$HOME/Downloads/av`）
2. aria2c 残留 → 启动前 `taskkill /f /im aria2c.exe`（Windows）或 `pkill aria2c`（Linux/macOS）
3. Windows 路径 → 始终用正斜杠 `/` 或双反斜杠 `\\`
