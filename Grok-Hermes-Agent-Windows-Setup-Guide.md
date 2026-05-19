![](https://files.mdnice.com/user/166515/f3182b55-0f5c-4c18-8eb4-6208558ae104.jpg)

> 趁着 xAI 把 Grok 订阅接进开源代理 Hermes 这条热点,这篇文章不谈观点,只带你在 **Windows 10 / 11** 上动手:从 PowerShell 安装到第一次对话,再到把它变成一个会定时干活的代理。全程不需要申请 API key,也不需要额外计费——你已经付费的 Grok 订阅就是入场券。

## 先说一句重要的话:原生 Windows 是早期 Beta

在动手之前,这一点必须摆在最前面,因为它会影响你的选择。

Hermes 现在可以在 Windows 10 / 11 上**原生运行**——不需要 WSL、不需要 Cygwin、不需要 Docker。但官方明确把原生 Windows 标记为 **early beta**:它能装能跑,也通过了 Windows 相关的检查,但没有像 Linux/macOS/WSL2 路径那样被大规模实测过。粗糙的地方主要集中在子进程处理、路径怪癖和非 ASCII 控制台输出上。

所以请按这个判断来选:

- **个人尝鲜、轻量 CLI、本地试用** → 原生 Windows 完全够用,按本文走
- **要长期挂 gateway、跑长任务、多平台消息接入做生产自动化** → 官方建议优先用 WSL2 或 Linux 服务器

本文走原生 Windows 路径;文末会给出何时该切到 WSL2 的明确信号。

## 你将得到什么

跟完这篇教程,你会在 Windows 上拥有一个本地运行的 Hermes Agent,它用你的 Grok 订阅做推理,能跨会话记事,并且可以挂一个定时任务在后台自动跑。整个过程大概 15 分钟。

需要先确认的前提:

- Windows 10 或 Windows 11(64 位;32 位会缺失 bash,功能受限)
- 一个**仍在生效的 Grok 订阅**(任意档位的 SuperGrok,或含 Grok 权益的 X Premium)
- 一个能打开网页的浏览器(本机即可)
- **不需要**预装 Python、Node.js 或 Git——安装器会自带一套隔离环境

> 关于依赖:Hermes 采用"零依赖"安装哲学。安装器会自行provision uv、Python 3.11、Node.js、ripgrep、ffmpeg,以及一份便携式 Git Bash(PortableGit,解包到 `%LOCALAPPDATA%\hermes`,不需要管理员权限,也不会碰你系统里已有的 Git)。如果你已经装了 Git,它会检测到并直接用。

## 第一步:用 PowerShell 安装 Hermes Agent

**用普通权限**打开 PowerShell(不需要"以管理员身份运行"),执行官方一行安装命令:

```powershell
irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1 | iex
```

![](https://files.mdnice.com/user/166515/b205883d-2116-4d5a-9392-1c3422c7c51e.png)

这条命令会把文件部署到 `%LOCALAPPDATA%\hermes\`,并把 `hermes` 加到你的**用户级 PATH**。

它在背后做的事大致是:装 uv、Python 3.11、Node.js、ripgrep、ffmpeg、便携 Git Bash,克隆仓库到 `%LOCALAPPDATA%\hermes\hermes-agent`,建虚拟环境,最后跑一遍首次设置向导(选模型、provider、工具集)。

**装完后最关键的一步,也是 Windows 上最容易踩的坑:** PATH 改动不会作用于已经开着的终端窗口。你必须**关掉当前 PowerShell,再开一个新的窗口**(或新开一个 Windows Terminal 标签页),`hermes` 命令才会生效。不要用 `$env:PATH += ...` 手工临时拼,除非你清楚自己在做什么。

新开窗口后验证安装:

```powershell
hermes --version
hermes doctor
```
![](https://files.mdnice.com/user/166515/4cc8d7f9-ec26-4694-81d8-b059675d3133.png)


`hermes doctor` 是这篇教程里你会反复用到的"体检命令"。它会列出环境、依赖和各个认证 provider 的状态。现在你应该能看到一个 `◆ Auth Providers` 区块,里面的 `xai-oauth` 还是未登录状态——这正常,下一步解决。

![](https://files.mdnice.com/user/166515/ef7ad822-2d50-42ae-ba69-6cff808240a9.png)

> 安全提醒:`irm ... | iex` 等于把远程脚本直接执行,和 Linux 上的 `curl | bash` 是一回事,值得谨慎。想先审一遍脚本,可以把那个 URL 在浏览器里打开读一遍,或者先 `irm <url> -OutFile install.ps1` 存下来检查后再运行。

> 如果遇到 PowerShell 执行策略阻止脚本运行,可在当前会话临时放开(仅作用于这个窗口,不改全局):`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`,然后重跑安装命令。

## 第二步:用 Grok 订阅登录(核心步骤)

这一步就是整条新闻的关键所在——浏览器一次授权,代替了传统的 API key 流程。

在新开的 PowerShell 窗口里走模型选择器:

```powershell
hermes model
```

接下来会发生这些事:

1. 从 provider 列表里选择 **"xAI Grok OAuth (SuperGrok Subscription)"**

![](https://files.mdnice.com/user/166515/e17f9767-0dce-4346-90c1-9012855e476b.png)

2. Hermes 自动打开你的默认浏览器,跳到 `accounts.x.ai`


![](https://files.mdnice.com/user/166515/bde8396a-6431-4067-a80b-af002de93973.png)


3. 你在浏览器里登录(或确认已登录的会话)并点击批准

![](https://files.mdnice.com/user/166515/f17d7d59-7444-444f-8ae1-701937f331b2.png)

4. xAI 重定向回 Hermes,令牌保存到 `%USERPROFILE%\.hermes\auth.json`
5. 回到选择器,挑一个模型——`grok-4.3` 永远被钉在列表最上面


![](https://files.mdnice.com/user/166515/249716ba-1d1f-4644-8b9c-4c762e7faac9.png)

6. 完成

如果你只想单独触发登录、不进模型选择器:

```powershell
hermes auth add xai-oauth
```

登录成功后,Hermes 会在每次会话前自动刷新令牌,你不用再管它,直到主动登出或在 xAI 账户设置里吊销授权。

再体检一次确认:

```powershell
hermes doctor
```

这次 `xai-oauth` 那一行应该显示为已认证。

![](https://files.mdnice.com/user/166515/ee935140-c7c1-4b27-be75-fb9f2e937d85.png)

> Windows 数据目录说明:`%LOCALAPPDATA%\hermes` 是可丢弃的基础设施(删掉后重跑安装命令即可恢复);而 `%USERPROFILE%\.hermes` 是你的数据——配置、记忆、skills、会话历史,结构和 Linux 安装完全一致。把这个目录在机器间同步,你的 Hermes 就跟着走。

> 常见小坑:浏览器授权有 180 秒超时窗口。点开浏览器后别去倒水了再回来,否则会看到 "Authorization timed out"。不要紧,重跑 `hermes auth add xai-oauth` 即可。

关于 SuperGrok 订阅: 这是本教程唯一的付费前提。值得注意的是,这次集成对所有订阅档位开放(xAI 未限制在高级套餐),且同一个登录就覆盖文本、语音、图像、视频、转写——相比单独申请 API key 还要管理速率和计费,订阅路径对个人用户更省心。推荐 **SuperGrok 官方订阅升级服务**:[cnmGrok.com](https://cnmgrok.com/)。
## 第三步:第一次对话

万事俱备,直接启动:

```powershell
hermes
```

![](https://files.mdnice.com/user/166515/1303564b-844f-478d-a0b3-a60c81cdc034.png)

随便问点什么,确认 Grok 真的在背后回应——让它做一道需要推理的题,或者解释一段代码。

![](https://files.mdnice.com/user/166515/54bdc159-c3c5-4672-8ec8-51d435289c27.png)

确认或固定模型可以这样设默认值:

```powershell
hermes config set model.default grok-4.3
hermes config set model.provider xai-oauth
```

设置完之后,`%USERPROFILE%\.hermes\config.yaml` 里会出现类似这样的内容:

```yaml
model:
  default: grok-4.3
  provider: xai-oauth
  base_url: https://api.x.ai/v1
```

到这里,你已经有了一个用 Grok 驱动、带长期记忆的本地代理。和普通聊天框的区别在于:它跨会话不丢上下文,而且能动手干活。

## 第四步:启用语音和图像(同一个登录,无需额外配置)

这次集成有个容易被忽略但很实用的点:同一个 OAuth 令牌不仅覆盖文本,还覆盖语音、图像、视频、转写,不用为每项能力再单独认证。

打开工具选择器:

```powershell
hermes tools
```

在菜单里给对应工具挑后端:

- **Text-to-Speech** → 选 "xAI TTS"
- **Image Generation** → 选 "xAI Grok Imagine (image)"
- **Video Generation** → 选 "xAI Grok Imagine"

如果 OAuth 令牌已存在,选择器会直接确认并跳过凭证输入。

> 提醒两点:第一,视频生成默认关闭,需要在 `hermes tools` 里进入 `🎬 Video Generation` 用空格键手动开启。第二,图像默认模型大约 5–10 秒出图,想要更高保真可以选 quality 版,代价是 10–20 秒。

## 第五步:让它在后台自动干活(Windows 上的关键差异)

Windows 关键差异: Linux 靠 cron / systemd;原生 Windows 上 gateway 跑成后台 PowerShell 进程,定时调度走 Windows 计划任务(Scheduled Task)。别处教程写 crontab 的地方,这里对应计划任务——照搬 Linux 教程最容易卡在这。
具体配置见官方文档:Windows (Native) Guide 的 "gateway as a Scheduled Task" 节,及 "Automate Anything with Cron" / "Daily Briefing Bot" 教程。三条纪律:

1. 先在 `hermes` 交互模式手动跑通,确认输出符合预期
2. 验证过的指令再固化成计划任务,别直接写没测过的调度
3. 要实时信息优先用 Grok 原生能力(模型层内置 X / 网络搜索,不必拼多步外部工具)

## Windows 常见问题速查表

教程类文章最有用的部分往往是出问题时怎么办。下面是针对 Windows 整理的高频情况:

1. **`hermes` 命令找不到 / 不是可识别的命令**
几乎都是因为没开新窗口。PATH 改动不作用于已打开的终端。解决:**关掉当前 PowerShell,开一个全新的窗口**,再 `Get-Command hermes` 验证。

2. **控制台中文/emoji 显示乱码**
原生 Windows 的非 ASCII 控制台输出是已知粗糙点。可以在环境变量里设 `HERMES_DISABLE_WINDOWS_UTF8=1` 回退到旧的 cp1252 stdio 路径(主要用于排查),或改用 Windows Terminal 而非老式 cmd 窗口。

3. **"No xAI credentials found"(运行时报找不到凭证)**
还没登录,或凭证文件被删了。解决:`hermes model` 选 xAI Grok OAuth provider,或直接 `hermes auth add xai-oauth`。

4. **令牌过期但没自动重新登录**
Hermes 会在每次会话前及遇到 401 时刷新令牌。若刷新令牌被吊销(你在 xAI 那边撤了授权,或账户轮换),它会给出明确的重新认证提示而非崩溃。解决:重跑 `hermes auth add xai-oauth`。

5. **"Authorization timed out"(授权超时)**
回环监听有 180 秒有效窗口,没及时批准就超时。解决:重跑 `hermes auth add xai-oauth` 或 `hermes model`。

6. **"State mismatch (possible CSRF)"**
Hermes 发现授权服务器返回的 `state` 与发出的不一致。解决:重新登录;反复出现就检查是否有代理或重定向在篡改 OAuth 响应。

7. **PowerShell 报执行策略错误,脚本无法运行**
当前会话临时放开:`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`,只作用于这个窗口。

8. **想彻底登出**
:`hermes auth logout xai-oauth`,
只想删某一条凭证池记录:先 `hermes auth list xai-oauth` 看清单,再 `hermes auth remove xai-oauth <index|id|label>`。

9. **想干净卸载整个 Hermes**
官方卸载路径会移除 `schtasks` 计划任务条目、启动文件夹快捷方式、`hermes.cmd` 垫片,删除 `%LOCALAPPDATA%\hermes\hermes-agent\`,并清理用户 PATH。注意:你的数据目录 `%USERPROFILE%\.hermes` 不在自动清理范围内,需要保留或手动删除自行决定。

## 什么时候该切到 WSL2

出现以下任一信号,转 WSL2:

- 需要 dashboard 网页内嵌终端面板(原生 Windows 无 POSIX PTY,仅此功能被禁,其余原生可用)
- 要做稳定的生产自动化(长任务、常驻 gateway、多平台消息)——官方明确建议优先 WSL2 / Linux
- 频繁遇到子进程、信号、路径分隔符的怪问题

切换:PowerShell 里 `wsl --install` 装 Ubuntu,在 WSL 里跑 `curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash`。两套可共存——原生数据在 `%LOCALAPPDATA%\hermes,WSL` 在 `~/.hermes`。

## 几条值得记住的实践纪律

1. 实时数据 ≠ 准确数据。 Grok 可能把过时或讽刺内容当事实。高风险决策(金融、法律、危机)行动前交叉验证。
2. 搜索成本会累积。 xAI 服务端搜索约 $5 / 千次;每天两三百次 ≈ $1/天,规模化要算进预算。
3. 凭证当密码对待。 令牌在 `%USERPROFILE%\.hermes\auth.json`,能花你的订阅额度。别打包上传,别留在不可信机器,登出真的跑 `hermes auth logout`。
4. 先手动跑通,再固化成计划任务。 摩擦降到只剩"点一下授权"后,决定成败的是你有没有先想清楚它会在哪出错。
