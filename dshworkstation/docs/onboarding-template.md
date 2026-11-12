# 接入一台新实例 —— 模板（先填值，再交给实例 Agent）

> 使用顺序：**先用「搭建工作台」文档（A）把工作台搭好**，你会得到域名、密语、证书三样东西；然后**把本文档里的占位符替换成这三样真实值**；最后把填好的本文档交给新 AutoDL 实例里的 Agent，让它照做即可。

## 〇、占位符替换表（动手前先填）

| 占位符 | 替换成 | 从哪来 |
|---|---|---|
| `YOUR_DOMAIN` | 你的域名，如 `mydsh.top`（**不带** www / https） | 文档 A「租域名」 |
| `YOUR_SECRET` | 你的工作台密语 | 文档 A「粘贴工作台代码」里设的 `SECRET` |
| `YOUR_CERT_PEM` | 你的 `cert.pem` **完整内容**（含 `-----BEGIN...` 和 `-----END...` 两行） | 文档 A「生成隧道证书」 |
| `__NODE_NAME__` | 本台实例名，如 `node2`、`node3`（唯一即可） | 自己起 |

> 填法：把本文档后面出现的这 4 个占位符**全部替换**成对应真实值，再交出去。

---

## 第 1 步：安装 cloudflared（GitHub 被墙走镜像）

```bash
wget -q https://gh-proxy.com/https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -O /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
cloudflared --version
```

## 第 2 步：安装 nginx

```bash
apt-get update -qq
DEBIAN_FRONTEND=noninteractive apt-get install -y -qq nginx
nginx -v
```

## 第 3 步：安装 dsh（⚠️ 先腾内存，否则被 OOM 杀）

容器有 **2 GiB 内存硬上限**，`npm install -g @deepseek-ai/dsh` 会额外吃 ~1.2 GB 触发 OOM。先停三个非关键服务，装完再启动：

```bash
supervisorctl -s unix:///tmp/supervisor.sock stop tensorboard jupyterlab autopanel 2>/dev/null || true
NODE_OPTIONS="--max-old-space-size=512" npm install -g @deepseek-ai/dsh --no-audit --no-fund
supervisorctl -s unix:///tmp/supervisor.sock start tensorboard jupyterlab autopanel 2>/dev/null || true
ln -sf "$(npm config get prefix)/bin/dsh" /usr/local/bin/dsh 2>/dev/null || true
dsh --version   # 应输出版本号
```

> 安装约 5–15 分钟属正常。若 `dsh --version` 找不到，用 `npm config get prefix` 找到 bin 目录，把里面的 `dsh` 软链到 `/usr/local/bin/dsh`。

## 第 4 步：写入隧道证书

把 `YOUR_CERT_PEM` 替换成 cert.pem 完整内容：

```bash
mkdir -p ~/.cloudflared
cat > ~/.cloudflared/cert.pem <<'CERT'
YOUR_CERT_PEM
CERT
chmod 600 ~/.cloudflared/cert.pem
```

> 证书是账号级凭证，所有实例共用。⚠️ 敏感，别公开。

## 第 5 步：创建本台实例的命名隧道 + DNS 记录

```bash
cloudflared tunnel create __NODE_NAME__
```

把输出里的**隧道 UUID** 填入下面再执行：

```bash
UUID=「这里填上一步输出的 UUID」
cat > ~/.cloudflared/config.yml <<EOF
tunnel: $UUID
credentials-file: /root/.cloudflared/$UUID.json
ingress:
  - hostname: __NODE_NAME__.YOUR_DOMAIN
    service: http://127.0.0.1:8080
  - service: http_status:404
EOF
cloudflared tunnel route dns __NODE_NAME__ __NODE_NAME__.YOUR_DOMAIN
```

## 第 6 步：写恢复脚本（把 `__NODE_NAME__`、`YOUR_DOMAIN`、`YOUR_SECRET` 都替换掉）

```bash
cat > /root/autodl-tmp/restart-dsh-open.sh <<'SCRIPT'
#!/bin/bash
set -e

echo "[1/4] 启动 cloudflared 命名隧道 __NODE_NAME__.YOUR_DOMAIN ..."
pkill -f "cloudflare[d] tunnel" 2>/dev/null || true
sleep 1
nohup cloudflared tunnel run __NODE_NAME__ > /tmp/cf-tunnel.log 2>&1 &
sleep 8
if ! pgrep -f "cloudflared tunnel run __NODE_NAME__" >/dev/null 2>&1; then
    echo "隧道启动失败，看 /tmp/cf-tunnel.log"; exit 1
fi
URL="https://__NODE_NAME__.YOUR_DOMAIN"
HOST="__NODE_NAME__.YOUR_DOMAIN"

echo "[2/4] 清理旧服务 ..."
pkill nginx 2>/dev/null || true
for p in $(ss -tlnp 2>/dev/null | grep -E ":6006|:8080" | grep -oP 'pid=\K[0-9]+' | sort -u); do
    kill "$p" 2>/dev/null || true
done
sleep 2

echo "[3/4] 启动 dsh web (trusted-host: $HOST) ..."
export NODE_OPTIONS="--max-old-space-size=512"
nohup dsh --profile web --port 6006 --trusted-host "$HOST" --no-open > /tmp/dsh-web.log 2>&1 &
sleep 6

echo "[4/4] 配置并启动 nginx ..."
rm -f /etc/nginx/conf.d/dsh-proxy.conf
cat > /etc/nginx/conf.d/dsh-proxy-open.conf <<'EOF'
server {
    listen 127.0.0.1:8080;
    server_name _;
    location / {
        proxy_pass http://127.0.0.1:6006;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
    }
}
EOF
nginx -t 2>&1 | tail -1
nginx

TOKEN=""
for _ in $(seq 1 90); do
    TOKEN=$(grep -aoE "token=[A-Za-z0-9_-]+" /tmp/dsh-web.log 2>/dev/null | head -1 | sed 's/token=//')
    [ -n "$TOKEN" ] && break
    sleep 1
done

echo "$URL/?token=$TOKEN" > /root/autodl-tmp/dsh_cur_web.txt

INSTANCE_NAME="__NODE_NAME__"
REGISTRY_URL="https://dsh.YOUR_DOMAIN"
REGISTRY_KEY="YOUR_SECRET"
if [ -n "$TOKEN" ] && [ -n "$REGISTRY_URL" ]; then
    curl -s -m 15 -G "$REGISTRY_URL/set" \
        --data-urlencode "name=$INSTANCE_NAME" \
        --data-urlencode "url=$URL/?token=$TOKEN" \
        --data-urlencode "key=$REGISTRY_KEY" >/dev/null 2>&1 \
        && echo "已上报工作台($INSTANCE_NAME)" || echo "警告: 上报失败(不影响本地访问)"
fi

echo ""
echo "======================================"
echo " 访问地址: $URL/?token=$TOKEN"
echo "======================================"
SCRIPT
chmod +x /root/autodl-tmp/restart-dsh-open.sh
```

## 第 7 步：跑一遍，接入工作台

```bash
bash /root/autodl-tmp/restart-dsh-open.sh
```

期望看到 `已上报工作台(__NODE_NAME__)` 和最终访问地址。

## 第 8 步：验证

```bash
# 本机 dsh web（期望 401）
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:6006/
# 经 nginx（期望 401）
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/
# 实例固定子域名带 token（期望 303）
curl -sk -o /dev/null -w '%{http_code}\n' "https://__NODE_NAME__.YOUR_DOMAIN/?token=<脚本输出的token>"
# 工作台应出现本实例
curl -s "https://dsh.YOUR_DOMAIN/json?key=YOUR_SECRET"
```

---

## 常见坑位

1. **npm install 被 OOM 杀**：第 3 步必须先停 tensorboard/jupyterlab/autopanel，并带上 `NODE_OPTIONS` 堆限制。
2. **GitHub 拉不动**：cloudflared 必须走 `gh-proxy.com` 镜像。
3. **新子域名解析慢**：`__NODE_NAME__.YOUR_DOMAIN` 建好后可能几分钟到几十分钟才解析，验证时别急着下结论。
4. **token 抓取**：dsh 首次启动可能 1–2 分钟才打印 token 行，脚本已轮询等 90 秒。
5. **工作台里点开报 `dsh web authentication required`**：只在实例用了 trycloudflare 随机域名（不同站）时发生；本模板用固定子域名（同站），不会。
6. **隧道起不来**：看 `/tmp/cf-tunnel.log`；常见原因是 config.yml 的 UUID 填错、或 cert.pem 没写对（漏了 BEGIN/END 行、或占位符没替换）。
7. **证书/密语泄露**：`cert.pem` 和密语都是敏感信息，别公开传播。
