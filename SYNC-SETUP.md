# Rudang-Quiz 同步中转（Cloudflare Worker）

前端只带一个"同步码"，这个 Worker 持有 GitHub Token，负责把做题记录写进你自己的
GitHub 仓库。Token 不会出现在网页里，也不受本地网络访问 `api.github.com` 的影响。

```
手机 / 电脑浏览器  ──►  Worker（持 Token）  ──►  GitHub 仓库 data/<同步码哈希>.json
   只有同步码
```

---

## 一、创建数据仓库（1 分钟）

1. 打开 https://github.com/new
2. 仓库名填 `Rudang-Quiz-Data`
3. 选 **Private**（记录是个人学习数据，建议私有）
4. **不要**勾 "Add a README file"，直接 Create repository

## 二、创建 Token（2 分钟）

1. 打开 https://github.com/settings/personal-access-tokens/new
   （Fine-grained token，不要用 classic，权限更小更安全）
2. Token name 填 `rudang-quiz-sync`
3. Expiration 选 1 年（到期换一个新的即可）
4. **Repository access** → 选 `Only select repositories` → 只勾 `Rudang-Quiz-Data`
5. **Permissions** → `Repository permissions` → 找到 `Contents` → 选 **Read and write**
6. 其他权限全部保持 No access，然后 Generate token

> 复制出来的 `github_pat_...` 只会显示一次，先存好，下一步要用。
> 这个 Token 只能读写 `Rudang-Quiz-Data` 一个仓库，丢了影响也有限。

## 三、部署 Worker（3 分钟）

1. 打开 https://dash.cloudflare.com/ 注册/登录（免费）
2. 左侧 **Workers & Pages** → **Create** → **Start from Hello World**
3. 名字填 `rudang-sync`（决定你的地址，记下来）
4. 点 **Deploy**，然后 **Edit code**
5. 把编辑器里的内容**全部删掉**，粘贴本目录 `worker.js` 的完整内容
6. 右上角 **Deploy** 部署

### 配置环境变量

在 Worker 页面 → **Settings** → **Variables and Secrets**，添加：

| 名称 | 值 | 类型 |
| --- | --- | --- |
| `GH_TOKEN` | 第二步拿到的 `github_pat_...` | **Secret**（加密） |
| `GH_OWNER` | 你的 GitHub 用户名，如 `JJJKK-OSS` | Text |
| `GH_REPO` | `Rudang-Quiz-Data` | Text |
| `GH_BRANCH` | `main` | Text（可省略，默认就是 main） |
| `ALLOW_ORIGIN` | `https://jjjkk-oss.github.io` | Text（建议填，防止别人盗用你的 Worker） |

保存后再 **Deploy** 一次让变量生效。

### 验证

浏览器打开：

```
https://rudang-sync.你的账号.workers.dev/health
```

看到 `{"ok":true,"service":"rudang-quiz-sync"}` 就通了。

## 四、在刷题页开启同步

1. 打开 https://jjjkk-oss.github.io/Rudang-Quiz/
2. 首页 → **☁️ 设置同步**
3. 服务器地址填 `https://rudang-sync.你的账号.workers.dev`
4. 同步码自己起一个（6 位以上，字母数字），或点**随机生成**
5. 点**保存并同步**

第一次开启时，会把本机已有的做题记录回填上传，所以另一台设备填**同样的地址和同步码**，
打开就能看到你已经做过的进度。

---

## 它是怎么合并的

- 每答一题产生一条事件：`{id, qid, dev, ts, ok, pick}`
- 服务端按 `id` 去重、按 `rev` 递增分配版本号
- 客户端只拉 `rev > 上次` 的增量，所以同步是增量的，不会全量重传
- 同一道题两端都做过，会**两边都记上**（本来就是两次作答），不会互相覆盖
- 断网时事件攒在本地队列，恢复后自动补传；浏览器关掉再开也会接着传

## 常见问题

**Q：同步码泄露了会怎样？**
别人可以读写你这份记录。改个新码就等于换了个新空间，旧数据还在仓库里，只是不再同步。

**Q：Worker 挂了会怎样？**
刷题完全不受影响，只是不同步。本地记录照常保存，恢复后自动补传。

**Q：想换设备 / 手机丢了？**
新设备填同样的地址和同步码即可，记录都在云端。

**Q：数据到底存在哪？**
你自己的 GitHub 私有仓库 `Rudang-Quiz-Data` 的 `data/` 目录下，一个同步码对应一个
JSON 文件，文件名是同步码的 SHA-256，别人拿到仓库也看不出是哪个码。

**Q：额度够不够？**
Cloudflare 免费额度每天 10 万次请求。一次同步 1～2 个请求，页面每 30 秒拉一次，
一天最多几千次，远远用不完。