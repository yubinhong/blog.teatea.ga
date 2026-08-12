---
title: GitLab 升级：按官方 Upgrade Path 逐版本升级
date: 2019-04-08 14:35:00
updated: 2026-08-12 22:30:00
tags:
  - GitLab
  - 运维
categories: gitlab
keywords: GitLab 升级 Upgrade Path 后台迁移 Linux Package Omnibus
description: 使用 GitLab 官方 Upgrade Path 工具规划升级路径，并在每个必经版本完成备份、健康检查和后台迁移。
---

GitLab 不能在任意版本之间直接跨级升级。跨越多个大版本时，必须先经过官方指定的必经版本（required upgrade stops），并在每一站等待后台迁移全部完成，再进入下一站。

本文适用于使用官方 Linux Package（以前称 Omnibus）安装的单节点 GitLab CE/EE。Docker、Helm、源码安装、多节点、Geo 和零停机升级的流程不同，应使用对应的官方文档。

> GitLab 会调整必经版本和条件。实际操作前必须重新使用官方工具计算，不能长期照搬某一条示例路径。

## 一、计算升级路径

打开 GitLab Support 团队维护的 [Upgrade Path 工具](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)，输入当前版本和目标版本，保存工具生成的完整路径。

先确认当前版本和版本类型：

```bash
sudo gitlab-rake gitlab:env:info

# Debian/Ubuntu
dpkg-query -W 'gitlab-ce' 'gitlab-ee' 2>/dev/null

# RHEL/AlmaLinux/Rocky Linux
rpm -qa | grep '^gitlab-\(ce\|ee\)'
```

规划路径时遵守以下规则：

1. 以 Upgrade Path 工具的实时结果为准。
2. 每个 `主版本.次版本` 都安装该系列最新补丁。例如路径要求经过 `18.5`，应安装最新的 `18.5.z`，而不是 `18.5.0`。
3. 不要执行不带版本号的 `apt install gitlab-ee` 或 `dnf upgrade gitlab-ee`，否则可能跳过必经版本。
4. 每到一个必经版本，都要阅读该版本到下一版本之间的 [Upgrade notes](https://docs.gitlab.com/update/versions/)。
5. 每一站的后台迁移未完成前，不得安装下一站。

从 GitLab 17.5 开始，必经版本采用 `x.2.z`、`x.5.z`、`x.8.z` 和 `x.11.z` 的节奏。旧版本还有额外或条件性停留点，不能只套用这个规律。

截至本文更新时，官方列出的常见必经版本如下：

| 大版本 | 常规必经版本 |
| --- | --- |
| GitLab 15 | `15.0.5`、`15.4.6`、`15.11.13` |
| GitLab 16 | `16.3.9`、`16.7.10`、`16.11.10` |
| GitLab 17 | `17.3.7`、`17.5.5`、`17.8.7`、`17.11.7` |
| GitLab 18 | 最新的 `18.2.z`、`18.5.z`、`18.8.z`、`18.11.z` |
| GitLab 19 | 最新的 `19.2.z`、`19.5.z`、`19.8.z`、`19.11.z` |

多 Web 节点、大型 CI 表、NPM Package Registry 或用户量较大的实例可能需要额外停留点。不要根据上表省略工具给出的条件性版本。目标版本尚未发布时，应停在当前可用且受支持的最新补丁版本。

## 二、升级前准备

### 1. 检查系统和组件

确认目标 GitLab 支持当前操作系统，并阅读路径中每个版本的升级说明。保证 PostgreSQL、Redis 和 Gitaly 正常运行，并预留足够磁盘空间。

```bash
sudo gitlab-ctl status
df -h
df -i
```

大型生产实例应先在生产环境副本中完整演练，记录每一站的安装和后台迁移耗时。

### 2. 执行健康检查

```bash
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rake gitlab:doctor:secrets
```

解决异常后再继续。还应测试登录、项目列表、Issue/Merge Request、Git clone/push 和 CI Job。

### 3. 确认后台迁移已完成

在“管理员 -> 监控 -> 后台迁移”中确认没有 Queued、Finalizing 或 Failed 任务。

较新版本可以使用 Rake task：

```bash
# GitLab 18.9 及以上
sudo gitlab-rake gitlab:background_migrations:list

# GitLab 18.8 及以下（如果当前版本提供）
sudo gitlab-rake gitlab:background_migrations:status
```

旧版本没有对应任务时，可以查询数据库：

```bash
sudo gitlab-psql -c "SELECT job_class_name, table_name, column_name, job_arguments FROM batched_background_migrations WHERE status NOT IN (3, 6);"
```

结果必须为零行。失败的迁移必须先修复和重试，不要直接在数据库中标记完成。

如果启用了 Elasticsearch/Advanced Search，还要检查搜索迁移：

```bash
sudo gitlab-rake gitlab:elastic:list_pending_migrations
```

### 4. 创建可恢复的备份

```bash
# 数据库、Git 仓库、上传文件等
sudo gitlab-backup create

# Linux Package 的配置和 secrets
sudo gitlab-ctl backup-etc
```

至少还要单独保存：

- `/etc/gitlab/gitlab.rb`
- `/etc/gitlab/gitlab-secrets.json`
- `/var/opt/gitlab/backups/` 中刚生成的备份

将备份复制到异机或对象存储，并验证恢复流程。GitLab 备份通常只能恢复到相同版本和相同版本类型（CE/EE）的实例，不能把旧版本备份直接恢复到新版 GitLab。

### 5. 安排维护窗口

暂停新的 CI/CD Pipeline 和 Job；生产环境可以开启 Maintenance Mode。现代 Linux Package 升级会自行停止和启动所需服务，不必像旧教程那样手动只停止 Unicorn、Sidekiq 和 NGINX。新版本已使用 Puma，旧的 Unicorn 命令不再适用。

## 三、逐站安装指定版本

假设工具给出的路径为：

```text
当前版本 -> STOP_1 -> STOP_2 -> TARGET
```

下面的命令每次只能填写下一站。`<version>` 要替换为完整版本号，例如 `17.11.7`；CE/EE 包名必须与当前实例一致。

### Debian/Ubuntu

```bash
# GitLab EE
sudo apt update
sudo apt install gitlab-ee=<version>-ee.0

# GitLab CE
sudo apt update
sudo apt install gitlab-ce=<version>-ce.0
```

查询实际可用版本：

```bash
apt-cache madison gitlab-ee
# 或
apt-cache madison gitlab-ce
```

### RHEL/AlmaLinux/Rocky Linux 8/9

```bash
# EL9，GitLab EE
sudo dnf install gitlab-ee-<version>-ee.0.el9

# EL9，GitLab CE
sudo dnf install gitlab-ce-<version>-ce.0.el9

# EL8 时将 el9 改为 el8
```

查询可用版本：

```bash
dnf --showduplicates list gitlab-ee
# 或
dnf --showduplicates list gitlab-ce
```

不要在一条命令中连续安装所有版本。每安装一站，都必须完成下一节的检查。

## 四、每个必经版本都要检查

```bash
sudo gitlab-rake gitlab:env:info
sudo gitlab-ctl status
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rake gitlab:doctor:secrets
```

同时测试 Web 登录、Git clone/push、Merge Request 和 CI/CD。

GitLab 的后台迁移由 Sidekiq 执行。安装完成不表示迁移已经结束，大型实例可能需要数小时甚至更久。只有所有迁移均为 Finished/Finalized，且没有 Failed 任务，才能进入下一站。

完整循环如下：

```text
安装下一必经版本
  -> 检查服务与业务功能
  -> 等待所有后台迁移完成
  -> 阅读下一站 Upgrade notes
  -> 备份或创建快照
  -> 安装下一必经版本
```

## 五、到达目标版本后的检查

```bash
sudo gitlab-rake gitlab:env:info
sudo gitlab-ctl status
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rake gitlab:doctor:secrets
sudo gitlab-backup create
```

最后确认：

1. 后台迁移全部完成。
2. 恢复 CI/CD Pipeline 和 Job。
3. 关闭 Maintenance Mode。
4. 将 GitLab Runner 升级到兼容版本。
5. 检查错误日志、Sidekiq 队列、磁盘、邮件、LDAP/SSO、Container Registry、Pages 和对象存储。
6. 记录每一站版本、起止时间、迁移耗时和异常处理过程。

## 六、失败与回滚

升级失败时不要继续安装下一版本，也不要仅靠降级软件包回滚。数据库 schema 可能已经改变，直接安装旧 RPM/DEB 可能进一步损坏实例。

正确做法是停止操作、保存日志，并按官方回滚文档将软件、数据库、仓库、配置和 secrets 一起恢复到升级前的同一版本。生产环境在没有经过恢复演练前，不应开始跨大版本升级。

## 官方资料

- [Upgrade Path 工具](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)
- [Plan your upgrade path](https://docs.gitlab.com/update/upgrade_paths/)
- [Before you upgrade](https://docs.gitlab.com/update/plan_your_upgrade/)
- [Upgrade Linux package instances](https://docs.gitlab.com/update/package/)
- [Check migrations before upgrade](https://docs.gitlab.com/update/background_migrations/)
- [GitLab upgrade notes](https://docs.gitlab.com/update/versions/)
- [Back up GitLab](https://docs.gitlab.com/administration/backup_restore/backup_gitlab/)
