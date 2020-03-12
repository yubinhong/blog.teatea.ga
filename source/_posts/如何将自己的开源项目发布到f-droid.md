---
title: 如何将自己的开源项目发布到f-droid
date: 2019-04-24 14:00:28
tags:
categories: f-droid
keywords: fdroid f-droid
description: 如何将自己的开源项目发布到f-droid
---
本人所在公司现在想要把公司的一个开源项目发布到f-droid，研究了一周，走了许多弯路，现记录于此，谨防遗忘。
### 1.注册一个gitlab.com的账号，fork fdroiddata项目，用于pull request。

### 2.安装fdroidserver（ubuntu 18.04）
``` bash
git clone https://gitlab.com/fdroid/fdroidserver.git
export PATH="$PATH:$PWD/fdroidserver"
cd $PWD/fdroidserver
./makebuildserver #这个会去下载android sdk,gradle等编译工具。
```
makebuildserver这个步骤会持续比较长的时间，之后会出现报错，这是因为在构建buildserver因系统环境导致的，这个可以跳过，不影响后续的步骤。

### 3.初始化
``` bash
git clone https://gitlab.com/xxx/fdroiddata.git
cd fdroiddata
fdroid init #create a base config.py and signing keys 
fdroid readmeta #Make sure fdroid works and reads the metadata files properly
```

### 4.导入开源项目
``` bash
fdroid import --url https://github.com/foo/bar --subdir app
```

### 5.配置项目的metadata
vim metadata/app.yml
``` bash
# F-Droid metadata template
#
# See http://f-droid.org/manual for more details
# and the Metadata reference
# https://f-droid.org/docs/Build_Metadata_Reference/
#
# Fields that are commented out are optional
#
# Single-line fields start right after the colon (with a whitespace).

Categories: (use those which apply)
 - Connectivity
 - Development
 - Games
 - Graphics
 - Internet
 - Money
 - Multimedia
 - Navigation
 - Phone & SMS
 - Reading
 - Science & Education
 - Security
 - Sports & Health
 - System
 - Theming
 - Time
 - Writing
License: (identifier from https://spdx.org/licenses)

# Website: (web link)
SourceCode: (web link)
# IssueTracker: (web link)

# Changelog: (web link)
# Donate: (web link)
# FlattrID: (number)
# LiberapayID: (number)
# Bitcoin: (bitcoin address)

Summary: (one sentence, no more than 30-50 chars, no trailing punctuation)
Description: |-
    Description of what the app does, starting on a new line. It should be as
    objective as possible and wrapped at 80 chars (except links and list
    items).

    A blank line means a line break, i.e. the end of a paragraph.

    Bulleted lists can be used:

    * Item 1
    * Item 2

    Links can be added like this:
    [https://github.com/org/project/raw/HEAD/res/raw/changelog.xml Changelog]

    Links to other apps too: [[some.other.app]]

RepoType: (git, git-svn, svn, hg or bzr)
Repo: (repo url, preferably https)

# At least one for new apps
Builds:
  - versionName: '1.0'
    versionCode: 1
    commit: v1.0
    subdir: app
    # submodules: true
    gradle:
      - yes
    # output: some.apk
    # prebuild: sed -i -e

# For a complete list of possible flags, see the manual

# MaintainerNotes: |-
#     Here go the notes to take into account for future updates, builds, etc.
#     Will be published in the wiki if present.

# The following options are described at this location:
# https://f-droid.org/docs/Build_Metadata_Reference/#UpdateCheckMode
AutoUpdateMode: (see the manual)
UpdateCheckMode: (see the manual)
CurrentVersion: (current version name)
CurrentVersionCode: (current version code, i.e. number)
```
修改相应内容。

### 6.编译测试
``` bash
fdroid readmeta #确认app.yml语法没有问题
fdroid rewritemeta app #将app.yml格式化
fdroid lint app #再次确认app.yml没有问题
fdroid build -v -l app #编译测试
P.S. 根据项目环境，可能需要安装相应的android ndk，配置完之后一般就没有了问题。
```
如果在编译过程中还有问题，可以查看fdroid [帮助文档](https://f-droid.org/zh_Hans/docs/)。
