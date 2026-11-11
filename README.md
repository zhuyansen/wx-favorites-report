# 微信收藏可视化 (wechat-favorites-viz)

从加密的微信 Mac 本地数据库中提取收藏数据，生成交互式可视化 HTML 报告。

[![Security-graded by Agent Skills Hub](https://agentskillshub.top/badge/zhuyansen/wx-favorites-report.svg)](https://agentskillshub.top/skill/zhuyansen/wx-favorites-report/)

## 效果预览

报告包含：统计仪表盘、月度趋势、类型分布、来源排行、活跃热力图、词云、标签云，以及可按类型/标签筛选的收藏浏览区。

> 截图请打开 `report.html` 后自行截取（数据含个人信息，不宜公开分享截图）

## 快速开始（macOS）

### 前置条件

- macOS (Apple Silicon 或 Intel)
- 微信 Mac 4.x 已登录
- Python 3.9+
- Claude Code（推荐，自动执行全流程）

### 方式一：用 Claude Code 一键执行

```bash
# 在 Claude Code 中说：
微信收藏可视化
```

Claude 会自动完成密钥提取、解密、解析、报告生成全流程。

### 方式二：手动分步执行

#### Step 1: 安装依赖

```bash
pip3 install frida frida-tools pycryptodome
```

#### Step 2: 准备签名副本

微信 App Store 版有 Hardened Runtime 保护，需要复制一份去掉签名限制：

```bash
cp -R /Applications/WeChat.app ~/Desktop/WeChat.app
codesign --force --deep --sign - ~/Desktop/WeChat.app
```

#### Step 3: 提取加密密钥

关闭当前微信，用 frida 启动桌面版微信：

```bash
killall WeChat 2>/dev/null; sleep 2

# 运行 frida hook 脚本（见 SKILL.md 中完整代码）
# 微信启动后：登录 → 打开「收藏」页面 → 等待 60 秒
```

frida 会 hook `CCKeyDerivationPBKDF` 函数，捕获所有 PBKDF2 密钥派生调用。

输出文件：`/tmp/wechat_frida_keys.log`

#### Step 4: 解密数据库

从 frida 日志中找到 favorite.db 对应的 enc_key（通过 salt 匹配），用 Python 解密：

```python
# favorite.db 路径:
# ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/
#   xwechat_files/<wxid>/db_storage/favorite/favorite.db

# 解密参数: SQLCipher 4
# AES-256-CBC, HMAC-SHA512, PBKDF2 256000 轮, page_size=4096, reserve=80
```

#### Step 5: 生成报告

```bash
# 解析数据
python3 parse_favorites.py --input favorite_decrypted.db --output data.json

# 生成 HTML 报告
python3 generate_report.py --input data.json --output report.html

# 打开（推荐用 http server）
cd <输出目录> && python3 -m http.server 8765
open http://localhost:8765/report.html
```

## 踩坑记录

在实现过程中经历了 6 轮迭代，以下是关键经验：

### 密钥提取（最难的部分）

| 尝试 | 方法 | 结果 | 原因 |
|------|------|------|------|
| 1 | C 内存扫描 x'hex' 格式 | 0 keys | WeChat 4.x 不用此格式存密钥 |
| 2 | 搜索原始 salt 字节 | 误匹配 | 匹配到 ASCII 字符串不是真密钥 |
| 3 | HMAC 暴力验证 (8B对齐) | 未找到 | 过滤条件跳过了真正位置 |
| 4 | 无过滤暴力 (4B对齐) | 未找到 | 密钥经 PBKDF2 派生后存储，不是原始 enc_key |
| 5 | DYLD hook / lldb | 被阻止 | macOS SIP + Hardened Runtime |
| **6** | **frida + CCKeyDerivationPBKDF** | **成功** | hook 系统 PBKDF2 函数捕获派生过程 |

### 关键踩坑

**坑 1: SIP 阻止签名 /Applications 下的微信**
复制到 ~/Desktop 再签名。

**坑 2: sudo 启动微信导致数据目录变成 /var/root**
不用 sudo，以普通用户身份运行 frida。

**坑 3: favorite.db 密钥在启动时不加载**
微信只在用户打开「收藏」页面时才加载 favorite.db。frida 运行期间必须手动打开收藏。

**坑 4: WeChat 4.x 表结构变化**
3.x 用 `FavItems` + `FavDataItem`，4.x 改为 `fav_db_item` 单表，content 是 XML。

**坑 5: XML 字段名不统一**
文章标题在 `<pagetitle>` 不是 `<title>`，聊天记录在 `<datalist><dataitem><datadesc>`，视频标题在 `<dataitem>` 内部的 `<datatitle>`。

**坑 6: JS 引号嵌套**
`onerror="this.style.display='none'"` 在 Python f-string + JS 模板字符串中会导致语法错误。用 `&quot;` 替代。

**坑 7: file:// 协议下 onclick 不工作**
改用 event delegation（addEventListener on parent element）。

**坑 8: ECharts 图表间距**
图表容器需要明确的 height，否则部分区域渲染为空白。

## 项目结构

```
~/.claude/skills/wechat-favorites-viz/
├── SKILL.md                    # Skill 定义（触发词、流程、参数）
└── scripts/
    ├── parse_favorites.py      # 解析器（SQLite/CSV/JSON → 统一 JSON）
    ├── generate_report.py      # 报告生成器（JSON → 单文件 HTML）
    └── demo_data.py            # 模拟数据生成（测试用）
```

## 报告功能一览

- 统计卡片：总数、跨越天数、日均、来源数
- 亮点发现：最忙日、最爱来源、主力类型
- 月度趋势：折线 + 面积图
- 内容类型分布：甜甜圈图
- 来源 Top 15：水平柱状图
- 活跃热力图：星期 x 小时
- 时段分布：小时柱状图、星期柱状图
- 词云：从标题 + 描述提取关键词
- 标签云：微信收藏标签
- 收藏浏览：按类型筛选 + 按标签筛选 + 全文搜索 + 排序 + 分页
- 详情弹窗：点击卡片查看完整内容、原文链接、来源、标签

## 已知限制

- 图片/视频/文件原始内容存在微信 CDN（加密格式），无法在报告中预览
- 文章缩略图来自微信公众号 CDN，需联网
- 密钥提取仅支持 macOS，需要 frida
- 每次微信更新后需要重新签名桌面副本

## 技术栈

| 组件 | 技术 |
|------|------|
| 密钥提取 | frida 17.x, CCKeyDerivationPBKDF hook |
| 数据库解密 | PyCryptodome AES-256-CBC, HMAC-SHA512 |
| 数据解析 | Python sqlite3 + regex XML |
| 可视化 | ECharts 5.x CDN, echarts-wordcloud 2.x |
| 报告格式 | 单文件 HTML，暗色主题，内联所有依赖 |
