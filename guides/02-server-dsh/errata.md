# 02 手册修订记录

> 对应文件：[`服务器选购搭建与dsh接入完全手册.pdf`](服务器选购搭建与dsh接入完全手册.pdf)。
> 当前版本：**2026-11-10 修订版**（24 页，页脚 1–21）。
> 如果你手上的 PDF 是 10 月版（21 页，页脚 1–18），**请重新下载**——那一版有多处命令错误会让部署直接失败。

本文件只记录「改了什么」，不再给改法。改法已经写进 PDF 正文。

核对方式：`@deepseek-ai/dsh@0.2.1-rc.5` 及其子包的发布产物（`lib/bin.js` 的命令与参数声明、`dsh-web-app` 的启动参数、`dsh-host-webserver` 的地址校验、`dsh-llm-deepseek` 的配置字段、`dsh-headless` 的用法示例）、PyPI 上 `deepseek-harness` 的项目说明、官方价格页。

---

## 2026-11-10 修订

### 会让部署直接失败（7 处）

| # | 位置 | 原文问题 | 本版处理 |
|---|---|---|---|
| 1 | 4.2 / 4.3 | tarball 装的 Node，npm 全局 prefix 指向 Node 自己的目录，`dsh` 落进不在 PATH 的路径，敲 `dsh` 报 command not found | 4.2 末尾加 `npm config set prefix /usr/local`，并补 `npx` 软链 |
| 2 | 4.2 | 往 `/usr/local` 写文件却没说明要 root | 段落开头注明「本段必须 root，普通用户先 `sudo -i`」 |
| 3 | 4.3 | `npm install -g @deepseek-ai/dsh` 不锁版本，latest（0.2.2-rc.x）要求 Node ≥ 24.9，装 Node 22.19 必然失败 | 改为 `@0.2.1-rc.5`，并写明不锁版本的后果 |
| 4 | 4.3 / 4.8 | 未提 pnpm。缺 pnpm 时任何 `dsh plugin` 命令直接报 `dsh: pnpm was not found` | 4.3 新增 pnpm 安装小节 |
| 5 | 4.4 | 环境变量文件名不存在（`LLM_API_KEY` 等），且写进 `/etc/environment` 后 systemd 读不到 | 改为 `/etc/dsh/dsh.env` + 单元文件加 `EnvironmentFile=`；变量只保留真实存在的 `DEEPSEEK_BASE_URL` |
| 6 | 4.5 / 4.6 | 4.5 以 root 试启动会在 DSH_HOME 生成 root 属主文件，4.6 只建目录不递归改属主，服务起不来 | 4.6 改 `chown -R dsh:dsh /var/lib/dsh` |
| 7 | 4.9 | Python SDK 示例里的 `HarnessAgent` / `run_task` 不存在（PyPI 上 `deepseek-harness` 是第三方 API 客户端，读法是 `DeepSeekHarness(...).chat(...)`） | 删掉 SDK 示例，改为子进程调 `dsh --profile headless` |

### 功能跑不通或方案不可行（3 处）

| # | 位置 | 原文问题 | 本版处理 |
|---|---|---|---|
| 8 | 4.8 / 附录 A | `dsh plugin add` 缺 `--profile`；且该子命令开了 `passThroughOptions()`，`--profile` 必须紧跟 `plugin` 之后 | 全部改为 `dsh plugin --profile web add ...` |
| 9 | 4.9 / 第五部分 / 附录 A | `dsh run` 不是子命令，`run` 会被当成 profile 名加载并报错 | 全部改为 `dsh --profile headless "任务"` |
| 10 | 4.7 | 三种远程方案都缺 `--trusted-host`，页面能开、`/api` 被 browser-trust 栅栏拦掉 | Tailscale 段补 `--trusted-host 100.x.x.x:3080` |
| 11 | 1.4 / 4.7 | Caddy 方案没说放行 80 / 443 | 1.4 安全组与 ufw 两处补齐 |
| 12 | 4.7 | 「公网 IP + 端口直连」物理上做不到：dsh 拒绝 `0.0.0.0` 通配地址，公网 IP 是 NAT 映射、网卡上没有 | 删除该行，标题改为「两种可行方式」，表格后补一段说明原因 |

### 会卡住但能自己试出来（3 处）

| # | 位置 | 原文问题 | 本版处理 |
|---|---|---|---|
| 13 | 4.5 | 「预期输出：本机访问 127.0.0.1:3080 出现界面」——`--host 127.0.0.1` 绑的是服务器回环，读者在自己电脑上必然打不开 | 预期输出改为 `curl -I` 服务器端验证；坑点 4 补 SSH 隧道做法 |
| 14 | 4.4 | 方式 A 说「启动后在设置里填 Key」，但 4.4 排在 4.5 启动之前，界面还不存在 | 改为「等 4.5 启动成功后回来配」 |
| 15 | 1.6 | `adduser deploy` 是交互式的，与「五步可整段复制」的承诺冲突 | 改为 `adduser --disabled-password --gecos "" deploy` |
| 16 | 4.10 | 只给了备份命令，没给恢复命令；root 解包出来的属主是 root，服务起不来 | 补恢复命令 + `--owner=dsh --group=dsh` + 兜底 `chown -R` |

### 表述与数据（3 处）

| # | 位置 | 原文问题 | 本版处理 |
|---|---|---|---|
| 17 | 4.6 坑点 5 | 「systemd 不继承 PATH，写 `dsh web` 会启动失败」——结论对，理由不准：systemd 有编译期默认搜索路径（通常含 `/usr/local/bin`） | 改为「用编译期固定目录，不读 shell 的 PATH 与 `.bashrc`，发行版间不一致，所以写绝对路径」 |
| 18 | 4.4 | 「deepseek-v4-pro 已并入 V4.1-Flash 计费」不成立，v4-pro 是独立模型独立计价 | 改为「独立计费，不在 Flash 的价格里」 |
| 19 | 附录 A | 升级命令 `npm update -g` 缺 sudo，且会升到 latest 破坏版本锁定 | 改为 `sudo npm install -g @deepseek-ai/dsh@目标版本号` |
| 20 | 4.11 | 排查表里「外网不通」那行还写着「临时直连只放行自己的公网 IP」，而直连方案在 4.7 已判为不可行 | 改为「按 4.7 打通 Tailscale 或 Caddy——dsh 不支持端口直连，别试」；同时补「command not found」与「pnpm not found」两条排查行 |

### 排版

- 4.6 单元文件首行 `Description=... web profile` 与下一行 `After=` 粘连，已分行。
- 4.7 表格删掉不可行的一行后，标题同步改为「两种可行方式」。

---

## 页码说明

页脚页码从正文第 1 页起算，封面与目录占掉前面 3 页：

```
阅读器显示的页号 = 页脚页码 + 3
```

引用位置时优先用**章节号**（如 `4.6`），或写成「阅读器第 N 页」。

---

## 验证路径

在一台全新 Ubuntu 22.04 上按顺序跑（`sudo` 已标注的按标注执行），全部通过才算部署正确：

```bash
node -v && npm -v                      # 1. v22.19.0
npm config get prefix                  # 2. /usr/local
which dsh                              # 3. /usr/local/bin/dsh
dsh -h                                 # 4. 打印帮助
pnpm -v                                # 5. 有版本号输出
dsh plugin --profile web --help        # 6. 不报 required option
dsh --profile headless "print 1+1"     # 7. 跑完打印结果并退出
curl -I http://127.0.0.1:3080          # 8. 200 或 302
sudo systemctl status dsh --no-pager   # 9. active (running)
ss -tlnp | grep 3080                   # 10. dsh 进程占用 3080
```

第 11 步是远程访问：手机连上 Tailscale 后访问 `http://100.x.x.x:3080`，界面能加载**并且能正常对话**（不是只有页面打得开）。
