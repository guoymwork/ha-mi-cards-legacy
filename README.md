> [!WARNING]
> # ⚠️ 本仓库已废弃（DEPRECATED）
>
> **请勿在新项目中使用本仓库。**
>
> 本仓库是米家风格设备面板的**第一代实现**（独立脚本 + SSH 部署），
> 已被 **HA 集成方案**完全取代：
>
> 👉 **https://github.com/guoymwork/ha-mi-cards**
>
> | | 本仓库（已废弃） | [新仓库](https://github.com/guoymwork/ha-mi-cards) |
> | --- | --- | --- |
> | 形态 | 独立 Python 脚本 | HA custom integration |
> | 运行位置 | HA 之外的常驻电脑 / NAS cron | **HA 进程内** |
> | 安装 | scp 脚本 + 配令牌 + 挂定时任务 | **HACS 一装即用** |
> | 凭据 | access_token（30 分钟过期）+ SSH | **零凭据** |
> | 更新 | cron 周期 | **注册表事件实时重建** |
> | 房间改名 | 需自建状态层维持 | `area_id` 天然稳定 |
> | 用户配置保护 | 无 | 有 |
> | 分发 | 拷贝脚本 | HACS 商店 |
>
> 两者**产品特性完全一致**（1.43:1 横向卡片、圆形状态按钮、实物图、
> 区域分类、设备详情浮层），从本仓库迁到新仓库时界面几乎不变。
>
> **本仓库保留的价值**：
> 1. 设计决策的来龙去脉（见 `docs/STATE-MODEL.md` —— 揭示了「用可变显示名当不可变标识」的坑）
> 2. 手写 WS 客户端、原子写状态、两段式图片流水线的实现参考
> 3. 批量迁移历史面板时的兜底通道
>
> **本仓库只做维护，不加新功能。** 新需求请到新仓库。

---

# 米家风格设备面板 · 第一代实现（独立脚本）

把小米智能家居那种「一个设备一张横向卡片」的桌面搬进 Home Assistant，
但所有逻辑跑在 **HA 之外** 的机器上。

> ⚠️ 再次提醒：这是废弃方案。请用 [ha-mi-cards](https://github.com/guoymwork/ha-mi-cards)。

---

## 它长什么样

```
┌────────────── 常驻电脑（开发机 / NAS） ──────────────┐
│                                                       │
│  refresh_tokens.py ──► 拿 access_token（30 分钟有效）  │
│                                                       │
│  ssh <nas-host>  cd /tmp/mi-cards                     │
│      │                                                │
│      ├─ pull_state.py ──► ha_states.json              │
│      │                   ha_registries.json           │
│      │                                                │
│      ├─ gen_dashboard.py ──► device_dashboard.json    │
│      │                     mi-missing-models.json     │
│      │                     mi-images.json              │
│      │                     mi-cards-state.json  ← 稳定 slug/icon 映射（跨版本持久） │
│      │                                                │
│      └─ deploy_device_dashboard.py ──► 写入 Lovelace 面板
│          bump_resource_version.py   ──► 资源 URL 加 ?v=时间戳，破 31 天缓存
│                                                       │
│  本地补图：fetch_raw_images.py（抓百科原图）            │
│            normalize_images.py（Pillow 归一化）        │
│            scp → 远端 www/mi-images/                 │
│                                                       │
│  入口：./deploy.sh（6 步幂等流水线，可挂 cron）          │
└───────────────────────────────────────────────────────┘
```

---

## 环境变量

**所有环境相关的配置都已参数化**，按自己的环境设置：

```bash
MI_WORK_DIR      工作目录（默认 /tmp/mi-cards）
MI_TOKEN_FILE    token 文件（默认 $MI_WORK_DIR/ha_token.txt）
MI_HA_HOST       HA 地址（默认 127.0.0.1）
MI_HA_PORT       HA 端口（默认 8123）
MI_HA_URL        HA 完整 URL（默认 http://ha.local:8123）
MI_WWW_DIR       HA 的 www 目录（默认 /config/www）
MI_IMG_DIRS      图片目录（逗号分隔）
MI_DASH_PATH     面板 URL path（默认 device-cards）
MI_STATE_FILE    稳定标识状态文件路径
MI_OVERRIDES     人工覆盖配置（区域图标/显示名）路径
MI_MISSING_FILE  缺图清单路径
MI_IMG_VER       图片版本号（破缓存）
MI_RESOURCE_NAME Lovelace 资源文件名（默认 mi-cards.js）

# 远端主机相关（deploy.sh / 抓图脚本用）
NAS_HOST          远端 HA 主机的 ssh 别名
NAS_WWW           远端 HA 的 www 目录
NAS_STATE         远端稳定标识状态文件路径
NAS_AUTH_PATH     远端 .storage/auth 路径（刷新令牌用）
MI_NAS_IMG_DIR     远端成品图目录
MI_NAS_TMP_IMG_DIR 远端临时图目录
```

---

## 运行

```bash
# 全流程（需要 ssh 别名指向远端 HA 主机）
NAS_HOST=my-nas \
NAS_WWW=/path/to/ha/config/www \
NAS_STATE=/path/to/ha/config/.mi-cards/state.json \
NAS_AUTH_PATH=/path/to/ha/config/.storage/auth \
MI_HA_URL=http://ha.local:8123 \
./deploy.sh

# 单阶段（需在远端主机上、工作目录内有中间产物时）
ssh <nas-host> "cd /tmp/mi-cards && MI_WORK_DIR=/tmp/mi-cards python3 sync.py --stage verify"
```

---

## 文件清单

### 主流程脚本

| 文件 | 行数 | 职责 |
| --- | --- | --- |
| `deploy.sh` | 72 | **唯一入口**。6 步流水线：刷令牌 → scp 脚本 → 拉快照 → 生成 → 补图 → 部署 → 校验 |
| `sync.py` | 231 | 阶段编排器。`--stage snapshot/generate/deploy/verify/all`，输出差分报告 |
| `ha_ws.py` | 131 | HA WebSocket 客户端，**纯标准库**（手写帧编解码），无第三方依赖 |
| `refresh_tokens.py` | 86 | 从远端 `.storage/auth` 取 refresh_token → 换 access_token → 写前端 localStorage 格式 |
| `pull_state.py` | 68 | 拉全部实体状态 + 实体/设备/区域三个注册表，落盘为 JSON |
| `gen_dashboard.py` | 931 | **核心**。状态 + 注册表 → Lovelace 面板配置；含型号识别、卡片选主、离线设备区 |
| `sync_state.py` | 406 | **稳定标识状态层**。区域/设备 ID → URL slug + 图标 的持久映射 |
| `deploy_device_dashboard.py` | 81 | 把生成的配置写进 Lovelace 面板 |
| `normalize_images.py` | 143 | Pillow 归一化产品图：裁主体 → 等比缩放 → 居中放 480×480 透明画布 |
| `fetch_raw_images.py` | 189 | 从米家产品百科抓**原图**（标准化会破坏原图，必须先存原图） |
| `fetch_baike_image.py` | 158 | 单型号调试用：抓一张图看看 |
| `bump_resource_version.py` | 54 | 给 Lovelace 资源 URL 的 `?v=` 换新值，破除浏览器缓存 |

### 实验/测试脚本（保留作回归参考）

| 文件 | 说明 |
| --- | --- |
| `test_nav_structure.py` | 校验生成的面板视图结构（视图数、path 唯一性） |
| `test_rename_adapt.py` | 模拟房间/设备改名，验证 slug 稳定 |
| `test_real_rename.py` | 真机改名回归 |
| `try_section_width.py` | sections 布局宽度实验 |

---

## 文件契约

所有中间产物都在 `$MI_WORK_DIR`（默认 `/tmp/mi-cards`）下，**阶段之间只靠这些文件通信**：

| 文件 | 生产者 | 消费者 | 内容 |
| --- | --- | --- | --- |
| `ha_token.txt` | `refresh_tokens.py` / `deploy.sh` | 所有需要访问 HA 的脚本 | access_token 纯文本 |
| `ha_states.json` | `pull_state.py` | `gen_dashboard.py` | `hass.states` 全量快照（list） |
| `ha_registries.json` | `pull_state.py` | `gen_dashboard.py` | `entity_registry` / `device_registry` / `area_registry` |
| `mi-cards-state.json` | `sync_state.py` | `gen_dashboard.py` | **稳定标识映射**（跨版本持久，见 `docs/STATE-MODEL.md`） |
| `mi-missing-models.json` | `gen_dashboard.py` | `fetch_raw_images.py` | 缺产品图的型号清单 |
| `mi-images.json` | `gen_dashboard.py` | `gen_dashboard.py` | 型号 → 实物图 URL |
| `device_dashboard.json` | `gen_dashboard.py` | `deploy_device_dashboard.py` | 最终 Lovelace 配置 |

`mi-cards-state.json` **不在** `$MI_WORK_DIR`，而在 `MI_STATE_FILE` ——
这是唯一必须跨版本存活的状态文件，其余都是可重算的中间产物。

---

## 已知脆弱点（也正是新仓库要解决的问题）

| 脆弱点 | 后果 | 新仓库的解法 |
| --- | --- | --- |
| 常驻电脑必须开机 | 电脑休眠/断网 → 面板停止更新 | 跑在 HA 进程内，HA 在就一直在 |
| 令牌 30 分钟过期 | 长跑任务中途失效 → 静默失败 | 用 HA 自己的 `hass` 上下文，不需要令牌 |
| SSH/SCP 依赖 | 换 HA 主机要改脚本 | 完全不需要 |
| 快照是「某一刻」 | 生成期间设备变化 → 配置与实际不一致 | 实时读注册表，无快照窗口 |
| 破缓存靠改资源 URL | 改一次刷一次，容易忘 | 资源 URL 带 `?v={VERSION}`，版本号 bump 即失效 |
| 面板覆盖无保护 | 冲掉用户自建面板 | `_has_foreign_content()` 保护 |
| 抓图在本机 | 本机没外网/Pillow 就缺图 | 在 HA 内抓，失败可重试并区分「未收录/请求失败」 |
| 直接读 `.storage/auth` | 能读配置目录就能拿管理员长期凭据 | **新仓库零凭据** |

---

## 延伸阅读

- `ai.prompt` —— 给 AI agent 的上下文说明（**改代码前必读**）
- `docs/ARCHITECTURE.md` —— 流水线逐阶段拆解、WS 协议、生成器内部逻辑
- `docs/STATE-MODEL.md` —— 稳定标识状态层的设计与数据结构
- `docs/ROADMAP.md` —— 继续维护本方案时的取舍

## 安全说明

本仓库**不包含任何凭据**。运行时需要的一切（HA 地址、ssh 别名、目录路径）
都通过环境变量传入。请不要把真实 token 提交进版本库。

## License

MIT —— 见 [LICENSE](LICENSE)。
