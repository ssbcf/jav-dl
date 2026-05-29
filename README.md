# jav-dl

JAV 番号查询与 BT 下载工具 — Claude Code Skill。

## 功能

- **查询模式** — 搜索番号对应的片名、女优、封面信息
- **下载模式** — 自动搜索 → 筛选 → BT 下载 → 清理

## 快速开始

### 1. 安装依赖

| 工具 | 用途 | 安装 |
|------|------|------|
| curl | HTTP 请求 | 系统自带 / `apt install curl` / `brew install curl` |
| aria2c | BT 下载 | `apt install aria2` / `brew install aria2` / `winget install aria2` |

### 2. 配置下载目录（可选）

默认下载到 `$HOME/Downloads/av`，可通过环境变量自定义：

```bash
export JAV_DL_DIR="/path/to/your/downloads"
```

### 3. 在 Claude Code 中使用

```
/load-skill jav-dl
```

然后：

| 你说 | 效果 |
|------|------|
| `查询 XXX-123` | 仅查询片名/女优/封面 |
| `下载 XXX-123` | 搜索并下载最佳版本 |

## 搜索来源

- [Nyaa (sukebei)](https://sukebei.nyaa.si/) — .torrent 文件 + 日文片名
- 磁力猫 — magnet 链接 + 中文标签

## 版本选择策略

```
无码(UC/U/破解) > 中文字幕(C) > 种子数多 > 文件小
```

## License

MIT
