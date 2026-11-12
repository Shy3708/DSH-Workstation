# dsh 实例工作台

一个域名，进入所有实例。

本仓库提供一套**可复制的搭建与接入模板**：用自有域名 + Cloudflare 把分散在各处的 dsh（DeepSeek Harness）实例收拢成一个入口页，点实例名字就能直接进，不用再记一堆临时地址、也不用反复复制 token。

> 仓库目前只包含文档与结构图，不含可直接运行的代码；工作台的 Worker 代码内嵌在搭建教程里。

---

## 这套东西是什么

最终形态是两级结构：

```
你（手机 / 电脑）
   │  打开 https://dsh.你的域名.com/?key=你的密语
   ▼
工作台网页（一个列表，列出所有实例，点名字即进）
   ├─ node1.你的域名.com  → 实例 1 的 dsh harness
   ├─ node2.你的域名.com  → 实例 2 的 dsh harness
   └─ ...
```

- **工作台**：一个网页，只负责列出所有实例及其在线状态，本身不跑任何 harness。
- **每台实例**：一台机器上跑的 `dsh`（DeepSeek Harness）+ 一条固定子域名隧道。

### 解决什么问题

- 实例分散在多台机器上，地址零散、易变，没有统一入口；
- dsh web 自带 token 认证，每次访问都要处理 token，手动复制粘贴很烦；
- 用临时隧道（trycloudflare）时地址每次重启都变，且属于「另一个站点」，从工作台点进去会被拦成 `authentication required`。

### 适合谁用

- 手上有**多台 AutoDL 实例**、都跑 dsh harness，希望从一个入口进入的人；
- 有自己的域名、愿意把域名托管到 Cloudflare 的人（不需要 ICP 备案，但需要实名）；
- 需要把这套东西交给别人复现的人 —— 本仓库的两份文档就是按「人照着搭、Agent 照着接」写的。

---

## 核心设计思路

三个约束直接决定了整套架构的形状，理解了就不会踩坑：

1. **dsh web 用 token 认证 + `SameSite=Strict` cookie。** 浏览器规定：从一个站点跳转到另一个站点时不发送 Strict cookie。所以实例必须和工作台**在同一个域名下**（用子域名），点进去才能正常登录。
2. **`*.workers.dev`、`*.pages.dev` 在国内被墙。** 工作台必须挂在自己的域名上，不能直接用 Cloudflare 的免费子域。
3. **快速隧道不适合当固定入口。** trycloudflare 免域名但地址随机、每次重启都变，且被视为不同站点。所以实例侧统一使用**命名隧道 + 固定子域名**。

由此得到的分工是：**域名与注册表在 Cloudflare 上（一处），隧道与 dsh 在每台实例上（多处）**，实例启动时把自己的访问地址上报给注册表，工作台只读注册表。

---

## 结构与角色

![工作台结构示意](docs/assets/architecture.png)

图（原图整理自手绘稿）表达的是这几层关系：

| 层 | 图 / 文档中的表述 | 对应到实现 |
|---|---|---|
| 入口层 | 自有域名：统一入口 | Cloudflare 上的工作台 Worker，绑定自定义域名 `dsh.你的域名.com`，用 `?key=你的密语` 查看 |
| 实例层 | Bridge 1 / Bridge 2 / … / Bridge N，每个标注 `AutoDL + DSH` | 每台 AutoDL 实例上的 dsh web + nginx + cloudflared 命名隧道，固定子域名 `nodeN.你的域名.com` |
| 能力边界 | 统一入口与实例选择 | 已实现的是统一入口与实例列表（含在线/离线探测）；跨实例调度尚未实现 |

> 说明：图中实例名为 `Bridge 1..N`，两份文档中统一用 `node1`、`node2` 这类命名。两者指同一层东西，接入时以文档里的 `__NODE_NAME__` 为准。

### 角色划分

| 角色 | 做什么 | 产出 |
|---|---|---|
| **搭建者（人）** | 租域名、接入 Cloudflare、部署工作台 Worker、生成隧道证书 | 域名、密语（`SECRET`）、`cert.pem` |
| **实例侧 Agent** | 在新实例里照《接入实例模板》执行，装 cloudflared / nginx / dsh，建隧道，起服务并上报 | 该实例的固定子域名与可访问地址 |
| **Cloudflare** | 托管 DNS、运行 Worker + KV 注册表、承载命名隧道 | 统一入口与注册表 |
| **实例内组件** | `dsh --profile web`（6006）→ nginx（8080）→ cloudflared 命名隧道 → 子域名 | 单个实例的访问链路 |

### 运行期目录与文件约定

以下路径来自接入模板，属实例内的约定（不是本仓库的目录结构）：

| 路径 | 用途 |
|---|---|
| `~/.cloudflared/cert.pem` | 账号级隧道证书，所有实例共用，权限 `600` |
| `~/.cloudflared/config.yml` | 本实例命名隧道的 ingress 配置 |
| `/root/autodl-tmp/restart-dsh-open.sh` | 恢复脚本：起隧道 → 起 dsh web → 配 nginx → 抓 token → 上报工作台 |
| `/root/autodl-tmp/dsh_cur_web.txt` | 脚本写入的当前访问地址（含 token） |
| `/tmp/cf-tunnel.log`、`/tmp/dsh-web.log` | 隧道与 dsh web 日志 |

本仓库自身结构：

```
.
├── README.md
└── docs/
    ├── deploy-guide.md          # 从零搭建工作台（原「搭建工作台教程」）
    ├── onboarding-template.md   # 接入一台新实例（原「接入实例模板」）
    └── assets/
        └── architecture.png     # 结构图（README 引用）
```

---

## 快速开始 / 部署步骤要点

完整步骤与全部代码见[搭建工作台教程](docs/deploy-guide.md)，这里是主干：

**准备**：一个域名（`.top`/`.icu` 首年几块，`.com` 约 ¥75/年）、一个免费 Cloudflare 账号、一台能跑 `cloudflared` 的机器。

1. **租域名**：阿里云 / 腾讯云购买，持有人选个人，**完成实名认证**。网站跑在 Cloudflare（境外），**不需要 ICP 备案**。
2. **接入 Cloudflare**：Add a domain → Connect a domain → 填裸域名 → Free 套餐 → 按提示把注册商的 NS 改成 Cloudflare 给的两个 NS，等域名变成 Active。
3. **部署工作台 Worker**：Workers & Pages 建 Worker → KV 建命名空间 `dsh-registry` → 以 **`REGISTRY`**（大小写必须一致）绑定到 Worker → 粘贴教程里的代码并改掉 `SECRET`。
4. **绑自定义域名**：Worker → Domains & Routes → Custom Domain，填 `dsh.你的域名.com`。
5. **验证**：打开 `https://dsh.你的域名.com/?key=你的密语`，应看到「dsh 实例工作台」+「还没有实例上报」。
6. **生成隧道证书**：装 `cloudflared`（GitHub 不通走 `gh-proxy.com` 镜像）→ `cloudflared tunnel login` → 浏览器授权 → 得到 `~/.cloudflared/cert.pem`。这份证书是**账号级凭证，所有实例共用**，妥善保存。

**做完这一步，你手上应该有三样东西：域名、密语、`cert.pem`。** 接下来才是逐台实例接入。

工作台对外只有四个路径，全部要用 `?key=` 校验：

| 路径 | 作用 |
|---|---|
| `/` | 实例列表页（含在线探测，页面每 60 秒自动刷新） |
| `/json` | 以 JSON 返回注册表内容 |
| `/set?name=&url=&key=` | 实例上报/更新自己的入口地址（URL 必须 https） |
| `/del?name=&key=` | 删除一个实例 |

---

## 接入新实例 / 新 Agent

完整步骤见[接入实例模板](docs/onboarding-template.md)，流程是「**先填值，再交给实例 Agent**」：

1. 按模板开头的占位符替换表，把 4 个占位符全部换成真实值：
   `YOUR_DOMAIN`（如 `mydsh.top`，不带 www / https）、`YOUR_SECRET`、`YOUR_CERT_PEM`（含 BEGIN/END 行的完整内容）、`__NODE_NAME__`（如 `node2`，唯一即可）。
2. 把填好的文档交给新实例里的 Agent，让它按第 1–8 步执行：

   | 步骤 | 内容 | 关键点 |
   |---|---|---|
   | 1 | 安装 cloudflared | GitHub 直连不通，走 `gh-proxy.com` 镜像 |
   | 2 | 安装 nginx | `apt-get install nginx` |
   | 3 | 安装 dsh | **先停 `tensorboard`/`jupyterlab`/`autopanel`**，容器 2 GiB 内存硬上限，否则 npm 安装会被 OOM 杀掉；带 `NODE_OPTIONS=--max-old-space-size=512` |
   | 4 | 写入隧道证书 | 写到 `~/.cloudflared/cert.pem`，`chmod 600` |
   | 5 | 建命名隧道 + DNS | `tunnel create __NODE_NAME__`，把 UUID 填进 `config.yml`，`tunnel route dns` 指到 `__NODE_NAME__.YOUR_DOMAIN` |
   | 6 | 写恢复脚本 | 起隧道 → 起 `dsh --profile web --port 6006 --trusted-host <host>` → nginx 8080 反代 6006 → 从日志轮询抓 token → 上报 `/set` |
   | 7 | 执行脚本 | 期望输出「已上报工作台」和最终访问地址 |
   | 8 | 验证 | 见下 |

3. 验证口径（模板给出的期望值）：本机 6006 返回 **401**、经 nginx 8080 返回 **401**、带 token 的固定子域名返回 **303**、工作台 `/json` 里能看到本实例。

几点必须知道的：

- **每台实例用同一个 `cert.pem`、同一个 `__NODE_NAME__` 只能用一个**；重名会覆盖注册表里的记录。
- **子域名解析可能几分钟到几十分钟才生效**，验证时不要急着下结论。
- 新增一台实例 = 复制模板 + 改 `__NODE_NAME__`，其余步骤完全一致；不需要改工作台代码。

---

## 常见坑位

| 现象 | 原因与处理 |
|---|---|
| `workers.dev` 打不开 | 国内被墙，必须用自有域名 |
| 从工作台点进实例报 `authentication required` | 实例与工作台不在同一域名下（用了 trycloudflare 随机域名），改用固定子域名 |
| 改 NS 后一直不生效 | 正常需几十分钟到几小时，核对 NS 与 Cloudflare 给的一致 |
| Worker 500 | 检查 KV 绑定名是否为 `REGISTRY`、自定义域名是否正确 |
| `npm install` 被 OOM 杀 | 先停非关键服务，并加 `NODE_OPTIONS` 堆上限 |
| 隧道起不来 | 看 `/tmp/cf-tunnel.log`；常见是 `config.yml` 的 UUID 填错，或 `cert.pem` 没写对（漏 BEGIN/END、占位符未替换） |
| 域名买了用不了 | 多半是没完成实名认证 |

---

## 文档索引

| 文档 | 给谁看 | 内容 |
|---|---|---|
| [搭建工作台教程](docs/deploy-guide.md) | **人** | 从零搭工作台：租域名 → 接入 Cloudflare → 部署 Worker → 绑定 KV 与自定义域名 → 生成 `cert.pem` → 坑位清单 |
| [接入实例模板](docs/onboarding-template.md) | **实例里的 Agent** | 占位符替换表 + 第 1–8 步接入流程 + 验证命令 + 坑位清单 |

使用顺序：先按搭建教程搭好工作台、拿到域名/密语/证书三样真实值，再把它们填进接入模板，最后把填好的模板交给实例 Agent。

---

## 分工与致谢

这套东西分两层，一层在实例内部，一层在实例之上。WryingY 完成了 DSH 在 AutoDL 上的部署，使得在任意设备的浏览器登录站点后，即可进入并操作对应 AutoDL 实例上的 DSH 工作台；本仓库作者构建了 workstation 这一层入口工作台，通过它可以从统一入口进入任意 AutoDL 实例所对应的 DSH。前一层解决单台实例怎么进得去，后一层解决多台实例怎么从一个地方进。
