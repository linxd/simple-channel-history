# 通道和代码模板历史记录

[![GitHub release](https://img.shields.io/github/v/release/diridium-com/simple-channel-history)](https://github.com/diridium-com/simple-channel-history/releases/latest)
[![License: MPL-2.0](https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg)](https://www.mozilla.org/en-US/MPL/2.0/)
[![Java 17+](https://img.shields.io/badge/Java-17%2B-blue)](https://openjdk.org/projects/jdk/17/)
[![OIE 4.7.0](https://img.shields.io/badge/OIE-4.7.0-blue)](https://github.com/OpenIntegrationEngine/engine)

一个 Open Integration Engine 插件，用于跟踪通道和代码模板的版本历史。

正在运行 OIE 4.6.0？请使用 [simple-channel-history 2.8.1](https://github.com/diridium-com/simple-channel-history/releases/tag/v2.8.1)。

## 功能特性

- 保存通道和代码模板时自动跟踪版本历史
- 分解式组件差异视图，带可导航树（脚本、连接器、过滤器/转换器步骤、插件属性）
- 并排差异查看器，支持词级内联高亮
- 颜色编码的变更指示器（已添加、已移除、已修改、未更改）
- 回退到任何先前版本
- 修剪旧版本以管理存储
- 已删除项跟踪：删除通道或代码模板时自动保存最终 XML 快照
- 数据库支持的存储（历史记录随数据库备份一起保留）
- **同时支持两种管理员**：经典 Swing 管理员和 OIE Web 管理员，功能完全对等

### Web 管理员

该扩展在 `webadmin/` 中包含 Web 管理员界面（由 Maven 构建到 `webadmin/web/plugin.js` 并打包到扩展 zip 中）。安装后，OIE Web 管理员会自动发现并添加：

- 在通道视图（右键点击通道或通道任务面板）和代码模板树上的**查看历史**操作，以及
- 设置中的**通道历史：已删除项**页签。

差异比较使用 Web 管理员基于 Monaco 的并排查看器；分解、变更检测和重排检测在浏览器中运行（引擎 Servlet 不变，向两种界面提供相同的原始修订 XML）。需要 OIE Web 管理员 4.6+（提供 `registerChannelAction` / `registerCodeTemplateAction` 扩展点和 `createDiffEditor`）。

<img src="https://raw.githubusercontent.com/wiki/diridium-com/simple-channel-history/images/4.jpg" alt="显示修订表的歷史对话框" width="500">

<img src="https://raw.githubusercontent.com/wiki/diridium-com/simple-channel-history/images/5.jpg" alt="带树导航和并排比较的分解式组件差异视图" width="700">

<img width="500" height="691" alt="image" src="https://github.com/user-attachments/assets/65577e17-7929-4a13-bded-a1248e97ed3c" />


## 支持的数据库

- PostgreSQL
- MySQL
- Oracle
- SQL Server
- Derby

## 构建

需要 Java 17+ 和 Maven 仓库（本地或远程）中的 OIE 库。

```bash
./scripts/install-engine-jars.sh
```

该脚本将本插件编译依赖的 4 个引擎 jar（`mirth-server`、`donkey-server`、`mirth-client-core` 和 `mirth-client`）安装到本地 Maven 仓库，取自与 POM 的 `mc.version` 匹配的已发布 OIE 版本，并根据 POM 中固定的 SHA-256（`oie.dist.sha256`）进行校验。CI 运行相同的脚本。每个引擎版本运行一次。

### 开发构建（未签名）

```bash
mvn clean package
```

### 发布构建（使用 YubiKey 签名）

签名构建需要带有代码签名证书的 YubiKey 和 OpenSC PKCS#11 库。

1. 将 `yubikey-pkcs11.cfg.example` 复制为 `yubikey-pkcs11.cfg` 并更新系统对应的库路径
2. 创建包含证书链的 `certchain.pem`（你的证书 + 中间 CA + 根 CA，PEM 格式）
3. 使用签名配置构建。`read -s` 读取 PIN 时不会回显，也不会将其留在 shell 历史记录中（bash 或 zsh）。该变量不会被导出，因此只有 Maven 能获得 PIN，而构建启动的进程（npm 及其运行的安装脚本）不会获得：

```bash
read -rs PIN
mvn clean package -Psigning -Dsigning.storepass="$PIN"
unset PIN
```

插件 zip 将位于 `package/target/simple-channel-history-<version>.zip`。

### 端到端测试

`e2e/run.py` 通过引擎自身的安装器将构建的 zip 安装到已发布的 OIE 引擎中，并在真实数据库上检查插件：全新安装、从 `e2e/upgrade-sources.txt` 中每个版本升级（为旧引擎构建的版本会先在该引擎上运行，引擎取自 `e2e/engines.txt`，然后在其下升级引擎）、卸载，以及与 [Role-Based Access Control](https://github.com/diridium-com/role-based-access-control) 并存安装（版本取自 `e2e/rbac-source.txt`），其中仅限某些通道的角色必须只能看到这些通道的已删除快照。CI 在每次推送和拉取请求时针对 Derby、PostgreSQL、MySQL、SQL Server（jTDS 和 Microsoft 驱动）以及 Oracle 运行，引擎分别运行在 Java 17 和 Java 21 上。当某个版本面向尚未发布的引擎时，CI 会使用 `--label-as` 在 POM 的 `mc.version` 引擎上运行它，即安装其 `plugin.xml` 改为标明该引擎的 zip 副本。

本地需要 Java 17 或更高版本、除 Derby 外每个数据库的 Docker，以及该 zip 所面向引擎的发行版 tarball（其 `plugin.xml` 中的 `mirthVersion`）。当该引擎为 POM 的 `mc.version` 时，`scripts/install-engine-jars.sh` 会在 `target/oie-dist/` 中保留检查过的副本；对整个项目执行 `mvn clean` 会删除它，因此之后需再次运行该脚本：

```bash
python3 e2e/run.py --db postgres --zip package/target/simple-channel-history-<version>.zip \
    --tarball target/oie-dist/oie_unix_4_7_0.tar.gz --work /tmp/sch-e2e-1
```

`--work` 目录必须尚不存在。每个引擎及其日志保留在那里。对于为尚未发布的引擎构建的 zip，请使用 `mc.version` 的 tarball 并像 CI 那样添加 `--label-as <mc.version>`。

## 安装

使用 OIE 管理员中的扩展管理器安装，或手动解压到 `extensions` 目录。安装后需要重启。

插件将在首次启动时创建必要的数据库表。

## 使用方法

### 查看历史
- 选择一个通道，在通道任务面板中点击"查看历史"
- 或右键点击通道并选择"查看历史"

### 比较版本
- 双击某个版本可将其与前一个版本进行比较
- 或选择两个版本（Ctrl/Cmd 点击）并点击"显示差异"

### 回退
- 在历史表中选择一个版本
- 点击"回退到所选"

回退通道会将其恢复为该修订版本中保存的状态：连接器、脚本、过滤器、转换器，以及初始状态和消息存储等通道属性。其启用或禁用状态、消息修剪设置和标签保持现状不变。

### 修剪
- 在历史表中选择一个版本
- 右键点击并选择"修剪旧版本"
- 所有比所选版本更旧的版本将被删除

## 设计说明

插件为何以这种方式工作、哪些是有意为之，以及测试覆盖了什么：[docs/design-notes.md](docs/design-notes.md)。

## 许可证

Mozilla Public License 2.0 (MPL-2.0)

基于 Kiran Ayyagari 的原始 [git-ext](https://github.com/kayyagari/git-ext) 插件。
