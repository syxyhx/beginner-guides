# 02 手册勘误

> 对应文件：[`服务器选购搭建与dsh接入完全手册.pdf`](服务器选购搭建与dsh接入完全手册.pdf)（21 页，2026 年 10 月版）。
> PDF 保持交付原样、不再改动，**修正以本文件为准**。改完源文件重新导出后，本文件同步删掉对应条目。

核对方式：`@deepseek-ai/dsh@0.2.1-rc.5` 与相关子包的发布产物（`lib/bin.js` 的命令与参数声明、`dsh-web-app` 的启动参数、`dsh-headless` 的用法示例）、官方价格页。页脚页码与阅读器页号的换算见文末。

---

## 一、会让部署直接失败

### 1. dsh 装完不在 PATH（4.2 / 4.3 / 4.6 / 4.8 / 4.9 / 附录 A）

4.2 用 tarball 装 Node，只把 `node`、`npm` 软链进 `/usr/local/bin`。tarball 版 npm 的全局 prefix 默认等于 Node 自己的安装目录，所以 `npm install -g @deepseek-ai/dsh` 生成的可执行文件落在 `/usr/local/node-v22.19.0-linux-x64/bin/dsh`——**这个目录不在 PATH**。

后果是连锁的：

| 位置 | 原文预期 | 实际 |
|---|---|---|
| 4.3 | `dsh -h` 打印帮助 | `command not found` |
| 4.6 第二步 | `which dsh` → `/usr/local/bin/dsh` | 无输出 |
| 4.6 单元文件 | `ExecStart=/usr/local/bin/dsh` | 路径不存在，服务起不来 |
| 4.8 / 4.9 / 附录 A | 各种 `dsh` 命令 | 全部不可用 |

**改法（二选一，推荐第一种）**

```bash
# 推荐：装 dsh 之前先把全局安装位置定到 /usr/local
npm config set prefix /usr/local
npm install -g @deepseek-ai/dsh
which dsh          # 应输出 /usr/local/bin/dsh
```

```bash
# 或者：装完补一条软链
ln -sf /usr/local/node-v22.19.0-linux-x64/bin/dsh /usr/local/bin/dsh
```

> **执行身份（原文本段没写清）**：4.2 的 `tar -xJf ... -C /usr/local`、`ln -sf ... /usr/local/bin`，以及上面两条改法，**都要写 `/usr/local`，都必须用 root 执行（或逐条加 `sudo`）**。如果按 1.6 的建议切到 `deploy` 用户跑这一段，会直接权限拒绝（`npm install -g` 报 `EACCES`）。整段用 root 是最省事的做法。

4.6 单元文件的 `ExecStart` 一律用 `which dsh` 的真实输出，不要照抄 `/usr/local/bin/dsh`。

### 2. `dsh plugin add / remove` 缺 `--profile`，直接报错（4.8 / 附录 A）

`plugin` 是子命令，它的 `--profile` 被声明为 **requiredOption**（源码里 `.requiredOption("--profile <name>", "the profile whose plugins to manage")`）。4.8 写的是：

```bash
dsh plugin add <插件包名>
dsh plugin remove <插件包名> --config.offline=true
```

两条都缺 `--profile`，CLI 会报「required option 缺失」并退出，插件根本装不上。

**还有一个更隐蔽的坑**：这个子命令开了 `passThroughOptions()`，`add` 之后的参数会被**原样透传给 pnpm**。所以 `--profile` 必须紧跟在 `plugin` 后面；写成 `dsh plugin add pkg --profile web` 时，`--profile web` 会被当成 pnpm 参数，同样没用。

**正确写法**

```bash
dsh plugin --profile web add <插件包名>
dsh plugin --profile web remove <插件包名> --config.offline=true
dsh plugin --profile web list
```

附录 A 的插件两行同样要改。顺带一提：4.8 里「查看当前 profile 启用了哪些插件」那行 `dsh plugin --profile web list` 本来就是对的，说明这处是笔误而非理解错误。

### 3. `dsh run` 不是子命令（4.9 / 第五部分 / 附录 A）

4.9 写：

```bash
dsh run --profile web "统计 /var/lib/dsh/workspace 下所有 md 文件的总字数，输出汇总报告"
```

`run` 并不存在。CLI 只认 `profile`（默认）、`plugin`、`dump-config`、`dump-config-schema` 这几条路径，`run` 会被当成位置参数丢给 web 应用，起的还是一个常驻 Web UI，**不会执行任务、不会打印结果退出**，headless 的目的完全达不到。

**正确写法**（`dsh-headless` 包里给出的官方示例就是这个形式）：

```bash
dsh --profile headless "统计 /var/lib/dsh/workspace 下所有 md 文件的总字数，输出汇总报告"
```

`headless` profile 还支持 `--json` 输出、`--session-id` 指定会话，也可以把任务从 stdin 读进来。同一处错误还出现在：

- 第五部分「组合玩法示例」：`cron 触发 dsh run` → 改成 `dsh --profile headless ...`
- 附录 A：`任务 dsh run --profile web "任务描述"` → 同上

### 4. 写进 `/etc/environment` 的变量，systemd 读不到（4.4 方式 B / 4.6）

原文写「写入环境变量文件（**之后 systemd 服务也会读取**）」，然后把密钥追加进 `/etc/environment`。这句不成立：`/etc/environment` 由 PAM 的 `pam_env` 读取，只作用于登录会话；systemd 启动服务不经过 PAM。而 4.6 的单元文件里只有 `Environment=HOME` 和 `Environment=DSH_HOME`，**没有 `EnvironmentFile=`**。

只影响选了「方式 B（环境变量）」的人：服务能起来，但拿不到密钥，一调模型就 401。选了「方式 A（Web UI 填 Key）」的不受影响。

**改法**：用独立文件，并在单元里显式指向它。

```bash
sudo install -d -m 750 -o root -g dsh /etc/dsh
sudo install -m 640 -o root -g dsh /dev/null /etc/dsh/dsh.env
sudo tee /etc/dsh/dsh.env >/dev/null <<'EOF'
LLM_API_KEY=sk-你的密钥
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-flash
EOF
```

`[Service]` 段加一行：

```ini
EnvironmentFile=/etc/dsh/dsh.env
```

再 `sudo systemctl daemon-reload && sudo systemctl restart dsh`。（`ProtectSystem=full` 会把 `/etc` 挂成只读，但读取 `EnvironmentFile` 不受影响。）

---

## 二、会让远程访问失败

### 5. 4.7 缺 `--trusted-host`（4.7 全部三种方式）

web profile 的启动参数是 `--host`、`--port`、`--public-url`、`--trusted-host <authority...>`（可重复）等。其中 `--trusted-host` 的定义是「the /api browser-trust fence accepts 的额外 authority」——也就是说，浏览器用非默认（非 loopback）的 Host 访问时，`/api` 请求需要 host 在信任列表里，否则被拦。

4.7 让人把 `--host` 改成 `100.x.x.x` 或套 Caddy，却没说 `--trusted-host`。结果是页面可能打得开、接口全挂，读者很容易误判成「防火墙没放行」，而这正是手册自己说的最难排查的一类问题。

**补法**

```bash
# Tailscale 组网
dsh web --host 100.x.x.x --trusted-host 100.x.x.x --port 3080

# Caddy 反代（反代到本机，但外部用域名访问）
dsh web --host 127.0.0.1 --trusted-host 你的域名 --port 3080
```

对应改 4.6 单元文件的 `ExecStart`，以及 4.7 里 Tailscale 那段的 `--host` 示例。

---

## 三、可能仍然拿不到密钥（建议实机确认）

### 6. `LLM_API_KEY` 未必是 dsh 认的变量名（4.4 方式 B）

4.4 方式 B 用的是 `LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL`。但在 `@deepseek-ai/dsh-llm-deepseek` 的发布产物里检索，出现的是 **`DEEPSEEK_API_KEY`**，没有 `LLM_API_KEY` 的引用。也就是说：即使按第 4 条把「变量读不到」修好了，**变量名本身不被识别，照样 401**。

各家 provider 插件通常各有自己的密钥变量名（`apiKeyEnv`），并没有一个统一的 `LLM_API_KEY`。

**建议**：这一条我还没在真机上跑通端到端，把握度中等。请在服务器上先用方式 A（Web UI → 设置 → 模型）填一次 Key，然后看 `$DSH_HOME` 下 `.credentials.yaml` 里实际写了什么字段——以那个为准改写方式 B。如果确实要用环境变量，把变量名换成对应 provider 的那一个（DeepSeek 是 `DEEPSEEK_API_KEY`）。

---

## 四、版本与数据

### 7. Node 版本与 latest 版 dsh 不匹配（4.2 / 4.3）

`npm install -g @deepseek-ai/dsh` 默认装 latest。当前 latest 是 `0.2.2-rc.1`，它的 `engines` 要求 **Node ≥ 24.9.0**；教程装的 22.19 会触发 `EBADENGINE` 警告，运行风险未知。

`0.2.1-rc.1` / `rc.2` / `rc.5` 都没有 `engines` 声明，22.19 可以跑。

**二选一，不要含糊**：

```bash
# 方案一：锁版本（改动最小，和本手册其余内容一致）
npm install -g @deepseek-ai/dsh@0.2.1-rc.5

# 方案二：升 Node（4.2 换成 24.9.0 的 tarball）
```

4.3 现在只写「建议锁定版本：`npm install -g @deepseek-ai/dsh@<版本号>`」却没给具体版本号，等于让人默认装 latest。建议直接写成 `@0.2.1-rc.5`。

### 8. 价格数字偏高（FAQ）

原文：V4.1-Flash 空闲时段输入 ¥1.5 / 输出 ¥4.5 每百万 tokens。

官方价格页（USD，每百万 tokens，空闲时段为峰值的一半）：`deepseek-flash` 缓存命中 $0.003、缓存未命中 $0.15、输出 $0.6。按 1 美元 ≈ ¥7.2 折算约为 **输入 ¥1.08 / 输出 ¥4.32 / 缓存命中 ¥0.02**。输出差得不多，**输入那一项偏高约 40%**。

顺带补一句原文没写的：缓存命中的输入只要 ¥0.02 左右，重复读同一份上下文时成本几乎可以忽略。

### 9. 「deepseek-v4-pro 已并入 V4.1-Flash 计费」不成立（4.4 模型名说明）

官方价格页上 `deepseek-v4-pro` 仍是**独立模型、独立计价**（缓存命中 $0.022、未命中 $0.66、输出 $1.98），和 `deepseek-flash` 并列，没有被并入。

真正被并入的是另一组旧名：`deepseek-v4-flash`、`deepseek-v4-flash-vision-exp` 已退役，请求由 V4.1-Flash 承接、按 Flash 价格计费。

---

## 五、排版

### 10. 末页整页空白

物理第 21 页（页脚标「第 18 页」）除页码外没有任何内容，是导出时多出来的一页。打印会白废一张纸，在源文件里删掉后重新导出即可。

> 因为这一页，PDF 物理共 21 页但只有 20 页有内容。

---

## 页码说明

本 PDF 的页脚页码从正文第 1 页起算，封面与目录占掉前面 3 页：

```
阅读器显示的页号 = 页脚页码 + 3
```

引用位置时优先用**章节号**（如 `4.6`），或写成「阅读器第 N 页」。

---

## 改完之后的验证路径

在一台全新 Ubuntu 22.04 上按顺序跑（**除标注 `sudo` 的都用 root**），全部通过才算真正修好：

```bash
node -v && npm -v                      # 1. v22.19.0
npm config get prefix                  # 2. /usr/local
npm config set prefix /usr/local       # 3. 没设过就先设
npm install -g @deepseek-ai/dsh@0.2.1-rc.5
which dsh                              # 4. /usr/local/bin/dsh
dsh -h                                 # 5. 打印帮助
dsh plugin --profile web --help        # 6. 不报 required option
dsh --profile headless "print 1+1"     # 7. 跑完打印结果并退出
sudo systemctl start dsh && systemctl status dsh --no-pager   # 8. active (running)
ss -tlnp | grep 3080                   # 9. dsh 占用 3080
```

第 10 步是远程访问：手机连上 Tailscale 后访问 `http://100.x.x.x:3080`，界面能加载**并且能正常对话**（不是只有页面打得开）。
