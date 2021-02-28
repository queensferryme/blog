---
title: "从 HomeBrew 迁移到 MacPorts"
date: 2021-02-24T17:38:38+08:00
---

在听取了一些朋友的建议后，我决定使用 [MacPorts](https://www.macports.org/) 代替 [Homebrew](https://brew.sh/) 作为 macOS 上的包管理器。这篇博客记录了我迁移的原因、过程和结果。

<!--more-->

## 原因

Homebrew 无疑是当下 macOS 上最流行的包管理器。举个例子，当我在 Google 上使用关键词「migrate from homebrew to macports」进行搜索时，出现的第一条结果却是 [Migrate from Macport to Homebrew](https://kobkrit.com/migrate-from-macport-to-homebrew-97da38c69f79)。

[![macports-homebrew-google-search.png](https://i.loli.net/2021/02/24/kHDOLmcpUW3PY7f.png)](https://i.loli.net/2021/02/24/kHDOLmcpUW3PY7f.png)

既然 Homebrew 如此流行，那么我又为什么要尝试迁移到 MacPorts 呢？整件事的起因其实相当简单而主观：Homebrew 核心开发者对用户社区的态度。他们在多个问题上未与用户充分讨论就作出了草率的决定，并且在事后无视大量用户强烈的反对意见继续执行，例如：

- 默认开启 Google Analytics，并且用户无法选择（opt-in）是否参与这项信息收集活动（参见[链接](https://github.com/Homebrew/brew/issues/142)）；
- 删除一些仍被不少用户使用的命令选项，并且事后表现出抗拒听取用户诉求（参见[链接](https://discourse.brew.sh/t/maintaining-a-tap-to-provide-options-for-a-formula/3871)）；

上面提到的两个例子中均出现了 Homebrew 社区管理员对用户回复进行编辑、隐藏、删除，乃至直接锁定整个讨论帖的情况。同样，在开发者 [Dominyk](https://github.com/DomT4) 退出 Homebrew [事件](https://github.com/Homebrew/brew/pull/4864)中，Homebrew 作者 [McQuaid](https://github.com/MikeMcQuaid) 先后修改并删除了 Dominyk 的一条回复，该回复则声称自己退出 Homebrew 是受到 McQuaid 的强迫。这种言论审查让我感动强烈的不适，尤其是在一个本该最自由的开源软件用户社区中。

如果你有使用 macOS Beta 版本，你也会[发现](https://github.com/Homebrew/brew/blob/6b2dbbc96f7d8aa12f9b8c9c60107c9cc58befc4/Library/Homebrew/os.rb#L20) Homebrew 在还未出现错误时就会直接告诉你：请不要向 Homebrew 报告你遇到的问题，因为你使用了未受支持的 macOS 版本，你应自己对一切后果负责。这种回应让我感到一种傲慢。

当然，很多人会说开发者志愿投入自己的时间和精力维护开源软件，用户在无偿使用的情况下没有资格向开发者要求什么 —— 如果你感到不爽，请放弃使用这个软件。我觉得这么说也没错。

> So, here I am.

## 过程

MacPorts 的[安装过程](https://www.macports.org/install.php)较 Homebrew 要复杂许多，这可能也是它不如 Homebrew 流行的原因之一。

首先你需要安装 Xcode。你可以通过 Mac App Store 下载，下载和安装都需要较长的时间，请耐心等待。

其次你需要在终端执行 `sudo xcodebuild -license` 来同意 Xcode 证书。然而你在这简单的一步也可能会遇到不少问题，比如：

```
xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance
```

这通常是因为 Developer Directory 默认位于 `/Library/Developer/CommandLineTools`，你需要执行 `sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer` 来切换，也可以通过 `xcode-select --print-path` 来查看。

如果你在使用 macOS Big Sur，请参考 <https://trac.macports.org/wiki/BigSurProblems>。

最后，你需要在 MacPorts 官网下载对应你 macOS 系统版本的 pkg 安装包。安装进度可能会卡在最后的位置不动，这是因为 MacPorts 正在使用 rsync 同步 port tree —— 包含了所有可以通过 MacPorts 安装的软件的信息的一个目录。这一步有些类似于 Ubuntu/Debian 上的 `apt update`，由于默认服务器位于国外，如果没有换源的话，同步速度会极其缓慢。但实际上这时 MacPorts 已经基本安装完成，你可以用活动监视器强制退出安装包进程，然后再配置透过国内镜像同步 port tree 即可。

进入 `/opt/local/etc/macports` 目录，编辑 `sources.conf`，删除里面的全部内容（不用担心，官方已经为你准备好了一份默认设置的备份在 `sources.conf.default`），并写入[^1]：

```
rsync://mirrors.tuna.tsinghua.edu.cn/macports/release/ports [default]
```

当然你也可以考虑同步 tarball 而非直接同步整个目录：

```
rsync://mirrors.tuna.tsinghua.edu.cn/macports/tarballs/ports.tar [default]
```

上面修改到的 `sources.conf` 配置文件控制 port tree 的同步，我们还需要再修改 `macports.conf` 来配置使用镜像加速 MacPorts 本身的更新。在 `macports.conf` 中找到以下两行：

```
#rsync_server        	rsync.macports.org
...
#rsync_dir           	macports/release/tarballs/base.tar
```

修改为（同样有备份在 `macports.conf.default`）:

```
rsync_server        	mirrors.tuna.tsinghua.edu.cn
...
rsync_dir           	macports/release/base
```

或者说如果你更喜欢通过 tarball 同步更新：

```
rsync_server        	mirrors.tuna.tsinghua.edu.cn
...
rsync_dir           	macports/release/tarballs/base.tar
```

如果你不喜欢使用镜像，也可以修改 `macports.conf` 中的以下几行配置以透过代理使用 MacPorts[^2]：

```
# Proxies. These have no default values. The analogous environment
# variables are "http_proxy", "HTTPS_PROXY", "FTP_PROXY", and
# "RSYNC_PROXY".
#proxy_http          	proxy1:12345
#proxy_https         	proxy2:67890
#proxy_ftp           	proxy3:02139
#proxy_rsync         	proxy4:11377
```

最后一步，执行 `sudo port -v selfupdate` 以同步 port tree 并将 MacPorts 更新到最新版本。

Zsh 用户还推荐安装 [zsh-users/zsh-completions](https://github.com/zsh-users/zsh-completions) 插件，里面包含了对 `port` 命令 Tab 补全的支持。

## 结果

一般来说，MacPorts 安装包会修改你的 `~/.zprofile` 文件（如果你使用 zsh 的话），通过 `export PATH="/opt/local/bin:/opt/local/sbin:$PATH"` 将 `port` 加入 `PATH`。

安装完成后，打开一个新的 Shell 会话，这时你应该已经可以使用 `port` 安装软件包了：

```bash
sudo port install wget
```

或者卸载：

```bash
sudo port uninstall wget
```

更酷的是，你可以连带所有依赖项一起卸载（Homebrew 官方不提供此功能哦😎）：

```bash
sudo port uninstall --follow-dependencies wget
```

列出已安装的软件：

```bash
port installed
```

列出已安装、并且是用户明确要求安装（而非依赖项）的软件（Homebrew 官方也不提供此功能哦😎）：

```bash
port installed requested
```

同步 port tree 并且升级 MacPorts 本身：

```bash
sudo port selfupdate
```

升级已安装但不在最新版本的软件：

```bash
sudo port upgrade outdated
```

清理缓存：

```bash
sudo port clean --all installed
```

在初步使用了 MacPorts 一段时间后，我能说它相比 Homebrew 确实还具有以下几点优势：

1. Homebrew 为了实现无 `sudo` 安装，在 `/usr/local` 目录下使用了许多 `chown` 和 `chmod`，甚至还带有 `-R` 参数，这点令我感觉不太安全；作为一个能在系统全局安装软件的包管理器，像 MacPorts 一样需要使用 `sudo` 安装的设计应该更合理一些。
2. MacPorts 对依赖树对管理似乎更完善，体现在卸载软件时可以使用 `--follow-dependencies` 把所有无用的依赖项一起卸载掉；Homebrew 目前没有这个功能，但有个第三方扩展 [homebrew-rmtree](https://github.com/beeftornado/homebrew-rmtree) 可以实现。
3. MacPorts 中每个软件包称作一个 port，有一个唯一的 [port tree](https://ports.macports.org/)；Homebrew 中每个软件包称作一个 formula，分布在各个 git 仓库里，默认只有 [homebrew-core](https://github.com/Homebrew/homebrew-core)，但也可以通过添加 tap (Third-Party Repositories) 来扩展。不考虑 tap 的话，MacPorts 的软件数量以大致 4:1 完胜 Homebrew。
4. MacPorts 的 port tree 形式上接近 AUR，管理机制也较为完善：每个 port 都有自己的页面，可以查看 maintainer/stats/ticket 等；同步 port tree 可以下载 tarball（有签名），也可以 rsync。 homebrew 的软件源管理方式就显得较为粗陋，只能通过 `git pull` 来同步。
5. MacPorts 的文档较 Homebrew 完善，社区支持也更友好。

当然 MacPorts 也有其劣势：

1. 用户界面相对不友好：Homebrew 输出通常都带有彩色高亮，MacPorts 则只输出可读性较差的纯文本。
2. 相对小众，用户社区体量小，因而对于问题的响应速度可能较慢。
3. 在 macOS Big Sur 上，MacPorts 和 [pyenv](https://github.com/pyenv/pyenv) 的合作不太好。即使在指定了 `LDFLAGS` 等链接参数的情况下，pyenv 依然会无视 MacPorts 安装的 openssl/readline 并自己从头编译一份（参见[链接](https://github.com/pyenv/pyenv/issues/1624)）。

最后值得一提的是，本文在 Homebrew 存在的问题与 MacPorts 和 Homebrew 的对比两部分大量参考了 Saagar Jha 的博客[^3]。Saagar Jha 曾是 Homebrew 的开发者，但现在也迁移到了 MacPorts，他/她在这个话题上应该很有发言权。其博客也应该比我的更值得一读，推荐给各位。

---

[^1]: MacPorts的安装, https://juejin.cn/post/6847902225033854983
[^2]: MacPorts behind a proxy, https://destefano.wordpress.com/2011/03/18/macports-behind-a-proxy/
[^3]: Thoughts on macOS Package Managers, https://saagarjha.com/blog/2019/04/26/thoughts-on-macos-package-managers/