# 02 手册勘误

> 对应文件：[`服务器选购搭建与dsh接入完全手册.pdf`](服务器选购搭建与dsh接入完全手册.pdf)（21 页，2026 年 10 月版）。
> PDF 保持交付原样、不再改动，**修正以本文件为准**。改完源文件重新导出后，本文件会同步删掉对应条目。

页脚页码与阅读器页号的换算见文末「页码说明」。

---

## 硬伤 1：dsh 装完不在 PATH，第 4 部分从 4.3 起全部卡住

**影响位置**：4.2（页脚 p9 / 阅读器 p12）、4.3、4.6 第二步、4.8、4.9、附录 A

**问题**

4.2 用 tarball 装 Node，只把 `node`、`npm` 两个软链进了 `/usr/local/bin`：

```bash
ln -sf /usr/local/node-v22.19.0-linux-x64/bin/node /usr/local/bin/node
ln -sf /usr/local/node-v22.19.0-linux-x64/bin/npm  /usr/local/bin/npm
```

tarball 版 npm 的全局 prefix 默认等于 Node 自己的安装目录，所以 `npm install -g @deepseek-ai/dsh` 生成的可执行文件落在：

```
/usr/local/node-v22.19.0-linux-x64/bin/dsh
```

这个目录**不在 PATH**，而 `/usr/local/bin` 里也没有 dsh。

**后果（一处断、后面全断）**

| 位置 | 原文预期 | 实际 |
|---|---|---|
| 4.3 验证安装 | `dsh -h` 打印帮助 | `command not found` |
| 4.6 第二步 | `which dsh` → `/usr/local/bin/dsh` | 无输出 |
| 4.6 单元文件 | `ExecStart=/usr/local/bin/dsh` | 路径不存在，服务起不来 |
| 4.8 插件管理 | `dsh plugin ...` | 同样不可用 |
| 4.9 headless | `dsh run ...` | 同样不可用 |
| 附录 A 命令表 | 全部 `dsh` 开头的命令 | 同样不可用 |

**正确做法（二选一，推荐第一种）**

```bash
# 推荐：装 dsh 之前先把 npm 的全局安装位置定到 /usr/local
npm config set prefix /usr/local
npm install -g @deepseek-ai/dsh
which dsh          # 应输出 /usr/local/bin/dsh
```

```bash
# 或者：装完之后补一条软链
ln -sf /usr/local/node-v22.19.0-linux-x64/bin/dsh /usr/local/bin/dsh
```

4.6 单元文件里的 `ExecStart` 一律用 `which dsh` 的**真实输出**，不要照抄 `/usr/local/bin/dsh`。

---

## 硬伤 2：写进 `/etc/environment` 的变量，systemd 读不到

**影响位置**：4.4 方式 B（页脚 p10 / 阅读器 p13）、4.6 单元文件

**问题**

原文写「写入环境变量文件（**之后 systemd 服务也会读取**）」，随后把 `LLM_API_KEY`、`LLM_BASE_URL`、`LLM_MODEL` 追加进 `/etc/environment`。

这句话不成立：`/etc/environment` 由 PAM 的 `pam_env` 模块读取，只作用于**登录会话**；systemd 启动服务不经过 PAM，不会加载它。而 4.6 的单元文件里只有 `Environment=HOME` 和 `Environment=DSH_HOME`，**没有 `EnvironmentFile=`**。

**后果**

只影响选了「方式 B（环境变量）」的人：服务能起来，但拿不到密钥，一调模型就 401。
选了「方式 A（Web UI 里填 Key）」的人不受影响——配置存在 `DSH_HOME` 里，systemd 重启照样读得到。

**正确做法**

改用独立的环境变量文件，不要写 `/etc/environment`：

```bash
sudo install -d -m 750 -o root -g dsh /etc/dsh
sudo install -m 640 -o root -g dsh /dev/null /etc/dsh/dsh.env
sudo tee /etc/dsh/dsh.env >/dev/null <<'EOF'
LLM_API_KEY=sk-你的密钥
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-flash
EOF
```

然后在 `/etc/systemd/system/dsh.service` 的 `[Service]` 段里加一行：

```ini
EnvironmentFile=/etc/dsh/dsh.env
```

再 `sudo systemctl daemon-reload && sudo systemctl restart dsh`。
（单元里的 `ProtectSystem=full` 会把 `/etc` 挂成只读，但读取 `EnvironmentFile` 不受影响。）

---

## 排版：末页整页空白

物理第 21 页（页脚标「第 18 页」）除页码外没有任何内容，是导出时多出来的一页。打印会白废一张纸，在源文件里删掉后重新导出即可。

---

## 页码说明

本 PDF 的页脚页码从正文第 1 页起算，封面与目录占掉前面 3 页，因此：

```
阅读器显示的页号 = 页脚页码 + 3
```

引用位置时请优先用**章节号**（如 `4.6`），或写成「阅读器第 N 页」，避免对不上。

---

## 修正后的最小验证路径

在一台全新的 Ubuntu 22.04 上按顺序跑一遍，六步全过就说明 4.2–4.6 的坑已绕开：

```bash
node -v && npm -v                 # 1. v22.19.0
npm config get prefix             # 2. /usr/local
npm install -g @deepseek-ai/dsh
which dsh                         # 3. /usr/local/bin/dsh
dsh -h                            # 4. 打印帮助
sudo systemctl start dsh && systemctl status dsh --no-pager   # 5. active (running)
ss -tlnp | grep 3080              # 6. dsh 进程占用 3080
```
