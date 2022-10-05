---
title: "🎉 Announcing the RSSerpent Project"
date: 2021-07-01T14:12:47+08:00
tags:
  - rss
---

大约在今年六月底时，我完成了本学期所有的课程设计，开始了悠闲的暑假生活。悠闲的暑假自然是要找点 side project 来做，于是一个在我脑海里盘桓已久的念头再度浮现了 —— 重写 [RSSHub](https://github.com/DIYgod/RSSHub)！说做就做，经过了一周左右的努力，我很高兴可以宣布 [RSSerpent](https://github.com/RSSerpent/RSSerpent) 项目已经正式发布 [0.1.0](https://github.com/RSSerpent/RSSerpent/releases/tag/0.1.0) 版本（[PyPI](https://pypi.org/project/rsserpent/0.1.0/)）了！项目目前尚未成熟，未来可能仍有许多变化，但我们已经实现了核心功能。我将在这篇博客里介绍发起 RSSerpent 项目的初衷、我对项目的设想、近况与未来的计划～

<!--more-->

---

***Q***：项目名字的由来？

***A***：因为本项目旨在成为 RSSHub 的替代品，为 RSS 爱好者们适配（大）部分不支持 RSS 的网站，所以我们希望有一个形如 「RSS-something」的名字。想到项目要使用 Python 编程语言，Serpent（蛇）这个单词看起来很合适，于是就 —— RSSerpent！

***Q***: The name of the projcet?

***A***: This project aims to become a free and open-source alternative to RSSHub, creating RSS feeds for websites that doesn't provide any. So we want a name like "RSS-something". Given that this project is to be written in the Python programming language, the word "Serpent" seems more than fit, hence the name -- RSSerpent!

---

***Q***：为什么要用 Python？为什么不用 TypeScript / JavaScript / Go 等？

***A***：原因如下：

1.  我喜欢。
2.  RSSerpent 的本质是一个爬虫程序。在这个领域内，Python 语言具有良好的生态。
3.  使用 Python 编写一些简单的 Web 服务很方便。
4.  我憎恨 JavaScript 和 Go，TypeScript 勉强尚可接受。

***Q***: Why use Python? Why not TypeScript / JavaScript / Go etc?

***A***: Reasons listed as follows：

1.  I like Python.
2.  RSSerpent is intrinsically a web scraper, for which Python's rich ecosystem could be an edge.
3.  It is easy to script a small web service in Python.
4.  I loathe  JavaScript & Go, TypeScript is mildly acceptable.

---

***Q***：这个项目是抄袭 RSSHub 的吗？

***A***：这个问题需要从两个方面来回答：

1.  从创意角度来说，是的。先有 2013 年开始的 RSS-Bridge 项目，后有 2018 年开始的 RSSHub 项目，没有这些前辈，我可能永远也想不到这个创意。
2.  从实现角度来说，基本不是。RSSerpent 是使用 Python 语言编写的，这就注定了 RSSerpent 与 RSSHub 的代码实现将会有根本性的不同。但作为一名 RSSHub 的贡献者，我在实现 RSSerpent 时又将无可避免地会部分借鉴前者。事实上，本项目创立的初衷就是为了解决我在 RSSHub 项目中发现的一些问题。

我将在 RSSerpent 项目 README 中明确向 RSSHub 致谢。考虑到 RSSHub 使用 MIT 开源协议，我认为上述行为是可以接受的。

***Q***: Is RSSerpent a copycat of RSSHub?

***A***: I shall answer this question from two aspects:

1.  Idea? Yes. From the RSS-Bridge project that was founded in 2013, to the RSSHub project that was founded in 2018, I could never come up with the idea without these brilliant predecessors.
2.  Implementation? Mostly no. RSSerpent is written in Python, thus destined to be greatly different from RSSHub (written in JavaScript). But as a contributor to RSSHub, I will inevitably refer to parts of its implementation. In fact, RSSerpent was created to solve some issues I saw in RSSHub in the first place.

I will specifically acknowledge RSSHub in the README. Given that RSSHub is open-sourced under the MIT license, I believe these would suffice.

---

***Q***：既然已经有了很流行的 RSSHub，为什么还要再写一个 RSSerpent 呢？

***A***：正如上文所说，RSSerpent 项目创立的初衷是为了解决我在 RSSHub 项目中发现的一些问题。具体来说有：

1.  路由机制不合理（参见[链接](https://github.com/DIYgod/RSSHub/issues/6691)）。目前 RSSHub 采用 monorepo 的项目结构，所有路由均存放在项目的 `/lib/routes/` 目录下。许多贡献者出于好心向 RSSHub 提交了一个新路由，然后可能就不再回来 —— 维护该路由的任务因而落在了少数几个活跃开发者身上。每天都有大量的新路由请求和老路由失效，这些路由相关的 issue 和 pr 耗尽了开发者和维护者们的精力，使得推动改进 RSSHub 核心功能极其困难。处理路由相关问题 -> 没有时间改进核心 -> 只能继续处理路由相关问题，这几乎形成了一个恶性循环。
2.  活跃开发者和维护者流失。创始人 DIYgod 目前出国留学、并且正专注于新项目 [RSS3](https://rss3.io/)，维护者 [NeverBehave](https://github.com/NeverBehave) 孤军奋战、维护项目十分辛苦，能够长期活跃帮助解决问题的贡献者也寥寥无几。人员的流失导致了问题的累积。此外，上文提到的不合理路由机制也可能是让开发者们热情耗尽的原因之一。
3.  缺乏类型检查。RSSHub 最初使用 JavaScript 编写，动态类型语言很灵活，但给项目长期维护带来了困难。尽管目前已有使用 TypeScript 重写RSSHub 核心功能的[动议](https://github.com/DIYgod/RSSHub/issues/7531)，但目前看来进展缓慢且困难重重。
4.  缺乏版本控制、没有发布流程（参见[链接](https://github.com/DIYgod/RSSHub/issues/7201)）。目前 RSSHub 项目没有合理的发布流程，所有用户均直接使用 `master` 分支的最新代码，如果 `master` 分支出现严重错误，所有自建实例均会不可用（例如 [`f09b8c4`](https://github.com/DIYgod/RSSHub/commit/f09b8c41c655cd63fa7a9a494379ed3432522ab9)）。
5.  此外还有一些不可避免的问题：项目经过较长时间的维护，出现部分复杂、不可读、难以维护的代码，成为了修复错误、重构项目、推进新功能时的严重阻碍。

这些问题并不是 RSSHub 维护者们的过错，我对他/她们耗费个人时间维护开源软件致以最高的敬意。但我希望能够吸取其经验，解决一些存在的问题，因此创建了 RSSerpent 项目。

***Q***: With the existing popular RSSHub project, why bother writing another one (RSSerpent)?

***A***: As mentioned above, RSSerpent was created to solve some issues I saw in RSSHub in the first place. Specifically, these issues are:

1.  Unreasonable mechanism for managing routers (see [link](https://github.com/DIYgod/RSSHub/issues/6691)). Currently RSSHub adopts a monorepo structure, with all routers residing in the `/lib/routes` folder. Many contributors come to RSSHub, submit a new router, and maybe never come back again. Therefore, it becomes the responsibility of a few core developers to maintain these routers. Everyday requests for new routers come in and old routers outdate, leading to numerous issues & pull requests that are exhausting to handle. These router-related issues have overwhelmingly outnumbered core-related issues, making it exceedingly difficult to push any advancements on RSSHub core functionalities.
2.  Loss of active developers & maintainers. Currently, the creator of RSSHub, DIYgod, has gone abroad for further study, with his focus shifted to the new [RSS3](https://rss3.io/) project. Another maintainer, [NeverBehave](https://github.com/NeverBehave), struggles to keep the project together. Little helpful active contributors are present and issues are accumulating. Also, the unreasonable routers machanism could be depleting the passion of developers (repeated, chore, trivial, tedious work).
3.  No type check. RSSHub was initially written in JavaScript, which is a flexible dynamically-typed programming language. However, this flexibility has caused issues for the project's long-term maintenance. Although there is already a [proposal](https://github.com/DIYgod/RSSHub/issues/7531) to rewrite RSSHub core in TypeScript, but advancements seem difficult to make.
4.  No proper version control & release schedule (see [link](https://github.com/DIYgod/RSSHub/issues/7201)).  Currently there is no proper release schedule in RSSHub. All users just directly use the code on the `master` branch. If something went seriously wrong on the `master` branch, all self-hosted instances could shutdown (e.g. [`f09b8c4`](https://github.com/DIYgod/RSSHub/commit/f09b8c41c655cd63fa7a9a494379ed3432522ab9)).
5.  Other inevitable issues: after quite some time, parts of the codes have become complex, unreadable, and hard to maintain. These codes become the major obstruction towards debugging, refactoring, and implementing new features.

Please note that these issues are not the fault of RSSHub maintainers. I pay tribute to them in the highest possible manner for maintaining this wonderful open-source software. However, I believe RSSerpent could be a beneficial attempt to learn from RSSHub's defects and solve these existing issues.

---

***Q***：你对项目的构想是什么？

***A***：RSSerpent 旨在成为 RSSHub 的一个替代品，因此其提供的功能与 RSSHub 基本一致。但我们想要提供一些新机制来解决 RSSHub 的问题：

1.  插件机制。路由将不再作为 RSSerpent 核心的一部分，而是可以按需加载的插件。每个插件都是一个独立的 Python 程序包，通过 `setuptools` entry points（[链接](https://packaging.python.org/guides/creating-and-discovering-plugins/#using-package-metadata)）来加载。用户在安装 RSSerpent 时，可以选择按需安装插件 —— 这些插件可以来自 PyPI，也可以直接来自某个 git 仓库。RSSerpent 官方则会维护部分较为流行、常用的插件。
2.  类型检查。项目 [type hints](https://www.python.org/dev/peps/pep-0484/) 覆盖，并使用 [mypy](https://github.com/python/mypy) 对项目源代码进行严格的类型检查。
3.  代码规范。使用 [black](https://github.com/psf/black) 和 [isort](https://isort.readthedocs.io/en/latest/) 格式化工具统一代码风格，[flake8](https://flake8.pycqa.org/en/latest/)（以及许多插件）对代码进行静态分析检查：包括注释、文档、命名规范、函数长度、代码复杂度等。
4.  测试覆盖。使用 [hypothesis](https://github.com/HypothesisWorks/hypothesis) 进行严格的单元测试，并且尽可能保证测试覆盖率 100%。
5.  版本发布。在 RSSerpent 核心，我们计划以较快的频率发布 alpha 测试版本（每 5 个 commit / 每三天），之后再逐渐发布 beta 测试版以及正式版 —— 我们想要确保每一个正式发布版本都是稳定且（尽可能）向后兼容的。此外，插件机制也给 RSSerpent 的版本控制带来了好处：对于某些变更频繁的路由，用户可以选择更激进的发布/更新策略，无需等待 RSSerpent 核心去合并 pr。
6.  工具链。RSSerpent 核心将会为各个插件提供一些常用的辅助工具函数，用于处理时区、图片懒加载、HTML 标签等等。此外我们将会引入插件[模板](https://github.com/RSSerpent/rsserpent-plugin-example)，使得用户可以通过 [cookiecutter](https://github.com/cookiecutter/cookiecutter) 快速创建并发布自己的插件。
7.  反-反爬虫机制。作为工具链的一部分，我们希望引入一些新的机制，来提升 RSSerpent 的反-反爬虫能力，降低路由的失效率。这些新反-反爬虫机制大致包括：HTTP Header 自动生成、缓存、速率限制、代理池等。
8.  完善文档。我们还希望通过引入一系列详尽的教程、文档，改善路由开发者的体验，尽可能降低路由开发者的准入门槛。理想状态下，拥有基本 Python 编程能力的人都能快速的写出自己的路由插件。此外，我们观察到（[链接一](https://news.ycombinator.com/item?id=24213682)，[链接二](https://www.reddit.com/r/rss/comments/ie2mxx/rsshub/)）许多 RSSHub 的非中文用户认为文档不详细、难以入门等，我们希望在这方面（i18N）也作出改进。
9.  赏金平台。RSSHub 也有通过 [IssueHunt](https://issuehunt.io/r/DIYgod/RSSHub) 使开发者在帮助解决问题时能获得小额收入，但目前来看使用者并不多。我们希望能在这方面作出增强，为新路由请求提供一个专有的的交流平台（[IssueHunt](https://issuehunt.io/)，[BountySource](https://www.bountysource.com/) 等），插件开发者在帮助缺乏编程知识的用户编写路由插件时，可以获得小额赞助，形成正向反馈。

***Q***: What is your overall design of RSSerpent?

***A***: RSSerpent is an alternative to RSSHub, providing mostly similar functionalities. However, some new mechanisms are to be introduced for solving issues in RSSHub.

1.  Plugins. Routers are no longer mandatory parts of RSSerpent core. Instead, routers are now plugins installed as demanded. Every plugin is an independent python package, loaded into the core by `setuptools` entry points ([link](https://packaging.python.org/guides/creating-and-discovering-plugins/#using-package-metadata)). When installing RSSerpent, users may choose to install plugins of their needs -- these plugins might be distributed from PyPI, or simply installed from some git repositories. RSSerpent will officially maintain some popular plugins.
2.  Type check. The RSSerpent project uses python [type hints](https://www.python.org/dev/peps/pep-0484/), and performs rigorous type checking with [mypy](https://github.com/python/mypy).
3.  Coding conventions. We use [black](https://github.com/psf/black) & [isort](https://isort.readthedocs.io/en/latest/) for unifying coding styles, as well as [flake8](https://flake8.pycqa.org/en/latest/) (and its many plugins) for static analysis: comments, docstrings, naming, functions, complexity are all checked.
4.  Test coverage. We use [hypothesis](https://github.com/HypothesisWorks/hypothesis) for property-based tests, and we try hard for 100% test coverage.
5.  Release. In RSSerpent core, we release alpha versions with a relatively high frequency (every 5 commit / every 3 day), succeeded by beta versions and stable versions . We want to ensure that every stable release is, like its name, stable and backward compatible. Moreover, the plugin mechanism has brought benefits: plugins can be released/updated at an independent pace, as they are decoupled from the core. You may choose a more radical release/update strategy, if the plugin is expected to frequently change.
6.  Toolchain. RSSerpent core provides common utility functions for processing timezones, lazy-loading, HTML tags etc. We also introduce a plugin [template](https://github.com/RSSerpent/rsserpent-plugin-example) so that developers could use [cookiecutter](https://github.com/cookiecutter/cookiecutter) to quickly scaffold & publish a new plugin.
7.  Anti-anti-bot. As part of the toolchain, we'd like to introduce something new so as to improve RSSerpent's anti-anti-bot capability. Currently we have these ideas in mind: HTTP header auto-generation, caching, rate limiting, proxy pool etc.
8.  Better documentation. We also would like to introduce a series of thorough tutorials & documents to improve developer experience and lower the barriers for beginners. Ideally, a person with basic Python programming skills should be able to write his/her own plugin. Also for international users, we have observed ([link1](https://news.ycombinator.com/item?id=24213682), [link2](https://www.reddit.com/r/rss/comments/ie2mxx/rsshub/)) common complaints about rough onboarding experience, so we are also working on better i18N.
9.  Bounty. RSSHub uses [IssueHunt](https://issuehunt.io/r/DIYgod/RSSHub), so that contributors could be rewarded for solving issues. However, very little RSSHub users post issues on IssueHunt. We believe it would be beneficial for RSSerpent to enforce a dedicated platform for requests of new routers, so that developers could be funded for creating new plugins, and people with less programming skills could also get the plugin they want in shorter time!

---

***Q***：项目目前进展如何？是否有近期计划或路线图？

***A***：项目目前已经完成第一阶段任务，发布 [0.1.0](https://github.com/RSSerpent/RSSerpent/releases/tag/0.1.0) 版本（[PyPI](https://pypi.org/project/rsserpent/0.1.0/)），基本实现了上述构想的 1～5 点。在第二阶段，我们将专注于上述构想中的 6～8 点。在第三阶段，我们将专注于第 9 点，提供更多官方插件，支持 [Atom](https://validator.w3.org/feed/docs/atom.html) / [JSON feed](https://www.jsonfeed.org/)，并通过 [XSL](https://www.w3.org/Style/XSL/) 支持在浏览器中直接阅读。我们将会以看板的形式将这写阶段目标细化并释出。

***Q***: What is the current status of RSSerpent? Is there any short-term plan or roadmap?

***A***: Currently we have finished Stage 1, released version [0.1.0](https://github.com/RSSerpent/RSSerpent/releases/tag/0.1.0) ([PyPI](https://pypi.org/project/rsserpent/0.1.0/)). Point 1-5, as mentioned above, were mostly completed. In Stage 2, we will focus on point 6-8. In Stage 3, we will finish point 9, create more official plugins, support [Atom](https://validator.w3.org/feed/docs/atom.html) / [JSON feed](https://www.jsonfeed.org/), and support reading in browser through [XSL](https://www.w3.org/Style/XSL/). The detailed plans of each stage will be released in a kanban.

---

目前需要说的就这么多。欢迎在这篇博客下方评论区留言，或加入 Telegram [群组](https://t.me/rsserpent)参与讨论！
