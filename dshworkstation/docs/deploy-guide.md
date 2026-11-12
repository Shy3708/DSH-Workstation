# 搭建「一个域名进入多个 dsh 实例」工作台 —— 从零教程（通用模板）

> 本文教**人**从零搭建工作台（域名 + Cloudflare + 工作台网页 + 隧道证书）。
> 完成后，再配合「接入实例」文档，即可让每台 AutoDL 实例自动接入本工作台。
> 本文是**通用模板**：文中的 `你的域名.com`、`你的密语` 等都要替换成你自己的值。

---

## 0. 这套东西最终是什么样

```
你（手机/电脑）
   │  打开 https://dsh.你的域名.com/?key=你的密语
   ▼
工作台网页（一个列表，列出所有实例，点名字即进）
   ├─ node1.你的域名.com  → 实例1的 dsh harness
   ├─ node2.你的域名.com  → 实例2的 dsh harness
   └─ ...
```

- **工作台**：一个网页，只负责列出所有实例，本身不跑任何 harness。
- **每台实例**：一台机器上跑的 dsh（DeepSeek Harness）+ 一条固定子域名隧道。

---

## 1. 为什么这么设计（关键原理，理解了才不会踩坑）

1. **dsh web 有 token 认证 + `SameSite=Strict` cookie**。浏览器规定：从一个网站点链接跳到**另一个网站**时，不发送 Strict cookie。所以实例地址必须和工作台在**同一个域名下**（子域名），点进去才能正常登录、不用复制粘贴。
2. **`*.workers.dev`、`*.pages.dev` 在国内被墙**，所以工作台必须挂在你自己的域名上。
3. **cloudflared 快速隧道（trycloudflare）**：免域名、随机地址，但每次重启变、且是「不同站」，不适合做固定入口。所以我们用**命名隧道 + 固定子域名**。

---

## 2. 需要准备什么

| 项 | 说明 | 花费 |
|---|---|---|
| 一个域名 | 任何后缀（`.top`/`.icu` 首年几块；`.com` ~¥75/年） | 几块~几十块/年 |
| 一个 Cloudflare 账号 | 免费 | ¥0 |
| 一台能跑 cloudflared 的机器 | 先随便一台（用来生成隧道证书） | — |

---

## 3. 第一步：租域名

国内最常用的是**阿里云（万网）**或**腾讯云**。

1. 打开 `wanwang.aliyun.com`（或腾讯云 DNSPod）。
2. 搜索想要的域名主体（如 `mydsh`），选一个便宜后缀（`.top`/`.icu`）。
3. 加入清单 → 结算：持有者类型选「个人」，年限 1 年，**增值服务能不要就不要**。
4. 支付宝/微信付款。
5. **完成实名认证**（付款后域名处于「未实名」，不实名会被暂停解析）。

> ⚠️ 关键：我们的网站跑在 Cloudflare（境外），**不需要 ICP 备案**。只需实名，无需备案。

---

## 4. 第二步：把域名接入 Cloudflare

1. 注册并登录 `dash.cloudflare.com`。
2. 点 **Add a domain / 添加站点** → 选 **Connect a domain**（连接，不是 Transfer 转移）。
3. 输入裸域名（如 `mydsh.top`，不带 www、不带 https://）。
4. 套餐选 **Free（$0）**。
5. 一路 Continue，最后 Cloudflare 会给你**两个 nameserver 地址**（形如 `xxx.ns.cloudflare.com`）。
6. 回到域名注册商（阿里云）→ 域名列表 → 管理 → 「DNS 修改 / 域名服务器」→ 改为「自定义 DNS」，填 Cloudflare 给的那两个 NS，保存。
7. 等生效（几分钟到几小时），Cloudflare 显示域名 Active。

---

## 5. 第三步：部署工作台 Worker（注册表）

### 5.1 创建 Worker
1. Cloudflare → **Workers & Pages** → **Create** → **Create Worker**。
2. 名字随便起（如 `dshworkbench`），先 Deploy 默认的。

### 5.2 创建 KV 命名空间
1. Workers & Pages → **KV** → **Create a namespace**，名字填 `dsh-registry`。

### 5.3 绑定 KV 到 Worker
1. 回到 Worker → **Settings** → **Variables and Secrets** → **Bindings** → **Add** → 选 **KV namespace**。
2. **Variable name 必须填 `REGISTRY`**（大小写一致）；KV namespace 选刚建的 `dsh-registry`。

### 5.4 粘贴工作台代码
点 **Edit code**，全选删除默认代码，粘贴下面完整代码，**把 `SECRET` 改成你自己的密语**：

```javascript
// dsh 实例工作台 —— 列出所有实例的入口页
// 部署：Cloudflare Worker，绑定一个 KV 命名空间为 REGISTRY
const SECRET = '改成你自己的密语'; // ← 查看和上报都用这个（?key=）

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const p = url.pathname.replace(/\/+$/, '');
    const key = url.searchParams.get('key');

    if (p === '/set') {
      if (key !== SECRET) return resp('unauthorized', 403);
      const name = (url.searchParams.get('name') || '').trim();
      const u = (url.searchParams.get('url') || '').trim();
      if (!name || !u) return resp('need name & url', 400);
      if (!/^https:\/\//.test(u)) return resp('url must be https', 400);
      await env.REGISTRY.put(name, JSON.stringify({ url: u, updatedAt: Date.now() }));
      return resp('ok ' + name, 200);
    }

    if (p === '/del') {
      if (key !== SECRET) return resp('unauthorized', 403);
      const name = (url.searchParams.get('name') || '').trim();
      if (!name) return resp('need name', 400);
      await env.REGISTRY.delete(name);
      return resp('deleted ' + name, 200);
    }

    if (p === '/' || p === '') {
      if (key !== SECRET) return resp('需要 key：在地址后加 ?key=你的密语', 403);
      const rows = await listAll(env);
      const withStatus = await Promise.all(
        rows.map(async (row) => ({ ...row, alive: await checkAlive(row.url) }))
      );
      return new Response(page(withStatus), {
        headers: { 'content-type': 'text/html;charset=utf-8' },
      });
    }

    if (p === '/json') {
      if (key !== SECRET) return resp('unauthorized', 403);
      const rows = await listAll(env);
      return new Response(JSON.stringify(rows, null, 2), {
        headers: { 'content-type': 'application/json' },
      });
    }

    return resp('not found', 404);
  },
};

async function listAll(env) {
  const list = await env.REGISTRY.list({ limit: 1000 });
  const rows = [];
  for (const k of list.keys) {
    const raw = await env.REGISTRY.get(k.name);
    try { rows.push({ name: k.name, ...JSON.parse(raw) }); } catch (_) {}
  }
  rows.sort((a, b) => a.name.localeCompare(b.name));
  return rows;
}

async function checkAlive(u) {
  try {
    const ctrl = new AbortController();
    const t = setTimeout(() => ctrl.abort(), 3000);
    await fetch(u, { method: 'GET', redirect: 'manual', signal: ctrl.signal });
    clearTimeout(t);
    return true;
  } catch (_) { return false; }
}

function resp(text, status) { return new Response(text, { status }); }

function esc(s) {
  return String(s).replace(/[&<>"']/g, (c) => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;',
  }[c]));
}

function page(rows) {
  const online = rows.filter((r) => r.alive).length;
  const items = rows.map((r) => {
    const ago = Math.max(0, Math.round((Date.now() - r.updatedAt) / 60000));
    const dot = r.alive ? '<span class="dot on"></span>' : '<span class="dot off"></span>';
    const st = r.alive ? '<span class="st on">在线</span>' : '<span class="st off">离线</span>';
    return '<li>' + dot +
      '<a href="' + esc(r.url) + '" target="_blank" rel="noopener">' + esc(r.name) + '</a>' +
      st + '<span class="m">' + esc(r.url) + '</span>' +
      '<span class="t">' + ago + ' 分钟前更新</span></li>';
  }).join('') || '<li class="empty">还没有实例上报</li>';

  return `<!doctype html><html lang="zh"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<meta http-equiv="refresh" content="60">
<title>dsh 实例工作台</title>
<style>
 body{font-family:-apple-system,system-ui,sans-serif;max-width:720px;margin:2rem auto;padding:0 1rem;background:#0f0f0f;color:#eee}
 h1{font-size:1.35rem;margin-bottom:.2rem} .sub{color:#777;font-size:.8rem;margin-bottom:1.2rem}
 ul{list-style:none;padding:0} li{position:relative;padding:.9rem 1rem .9rem 2.4rem;margin:.6rem 0;background:#1b1b1b;border-radius:10px}
 a{color:#7aa2f7;font-size:1.12rem;text-decoration:none}
 .dot{position:absolute;left:1rem;top:1.1rem;width:9px;height:9px;border-radius:50%}
 .dot.on{background:#3fb950}.dot.off{background:#f85149}
 .st{float:right;font-size:.72rem;padding:.15rem .5rem;border-radius:10px}
 .st.on{background:#12331c;color:#3fb950}.st.off{background:#3a1215;color:#f85149}
 .m{display:block;font-size:.72rem;color:#888;word-break:break-all;margin:.25rem 0}
 .t{font-size:.68rem;color:#666} .empty{color:#666}
</style></head><body>
<h1>🤖 dsh 实例工作台</h1>
<div class="sub">${rows.length} 个实例 · ${online} 在线 · 页面每 60 秒自动刷新</div>
<ul>${items}</ul>
</body></html>`;
}
```

点 **Deploy**。

### 5.5 给 Worker 加自定义域名
1. Worker → **Settings** → **Domains & Routes** → **Add** → **Custom Domain**。
2. 输入 `dsh.你的域名.com`（推荐用 `dsh.` 子域；直接用裸域名也行）。
3. 保存。

### 5.6 验证工作台
浏览器打开 `https://dsh.你的域名.com/?key=你的密语`，应显示「dsh 实例工作台」+「还没有实例上报」。

---

## 6. 第四步：生成隧道证书（cert.pem）

这份证书是账号级凭证，**所有实例共用**。在一台装了 cloudflared 的机器上：

1. 安装 cloudflared（GitHub 直连不通时走镜像）：
   ```bash
   wget -q https://gh-proxy.com/https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -O /usr/local/bin/cloudflared
   chmod +x /usr/local/bin/cloudflared
   ```
2. 发起登录：
   ```bash
   cloudflared tunnel login
   ```
3. 它会打印一个 `dash.cloudflare.com/argotunnel?aud=...` 的链接，**在已登录 Cloudflare 的浏览器里打开**，选择你的域名，点 **Authorize / 授权**。
4. 授权成功后，机器上会生成 `~/.cloudflared/cert.pem`（有时浏览器也会下载一份，两者内容一样）。
5. **保存好这份 `cert.pem`**——之后每台实例接入时都要用到它（内容直接内嵌进「接入实例」文档）。

---

## 7. 完成后的产物

搭建完成后，你手上应该有：

1. 一个域名（如 `mydsh.top`，已接入 Cloudflare）；
2. 工作台入口 `https://dsh.mydsh.top/?key=你的密语`；
3. 一份隧道证书 `cert.pem`；
4. 一个 KV 里还没有任何实例的空工作台。

接下来，就是**每台实例的接入**——这部分交给「接入实例」文档（含上面这些真实值），让每台实例的 Agent 照着执行即可。

---

## 8. 常见坑位

1. **`workers.dev` 打不开** → 国内被墙，必须用自有域名（本教程已用）。
2. **从工作台点进实例报 `authentication required`** → 实例和工作台不在同一域名下（用了 trycloudflare 随机域名）；改用固定子域名即可。
3. **改 NS 后一直不生效** → 正常等几十分钟到几小时；核对 NS 与 Cloudflare 给的一致。
4. **Worker 500** → 检查 KV 绑定名是不是 `REGISTRY`、自定义域名加对没。
5. **`cloudflared tunnel login` 证书没下载** → 看浏览器有没有下载 `cert.pem`；若机器上 `~/.cloudflared/cert.pem` 已生成则成功。
6. **域名买了用不了** → 可能没完成实名认证。
