---
title: CVESearchAndSendJIRA 源码学习笔记
date: 2026-07-09
tags: [CVE, Python, wxPython, 源码学习, 实习]
---

## 概述

FOSS 组件清单和本地漏洞数据库做匹配，结果写入统一 Excel，并准备转换导入格式。

## main.py

创建文本框，添加两个分页面 CVE和JIRA，其中两个UIbase父类具体在mainUI实现

## mainUI.py

先是工具包装，包括字体，颜色，控件初始化。构建布局，有整体布局包含顶部 左边 右边。随后是文件路径搜索，文件夹路径搜索，用os.walk的方法遍历查找。读取用户选取状态到Userconfig全局保存变量，在后续检测用户填入数据完整性（run之前需要检测）与文件路径检测配合确定可以开始thread后台运行，确保用户关键输入和选取文件正确。run_woker中出现的函数中用到WebsiteHandle去处理回调进度，出自调度层WebsiteHandle.py文件

## WebsiteHandle.py

感觉像是一个中间承接的感觉，上关联至UI层，用户在UI层点击按钮触发事件后，在调度层进行调用解析，随后传到parsers同名的函数中开始解析各条漏洞CVE，调用在FileOperation.py的函数执行写入EXCEL等操作。然后是过程记录的薄封装，包含了用回调显示进度。

NVD的JSON单独处理，与FOSS组件匹配后写入excel，是异步数据处理函数。iter_matching_records parse_nvd_item asyncio.to_thread，避免同时调用线程，防止阻塞。run_files异步处理文件，设置并发数，读取成list结果返回至excel。

## parsers：common

展示了最终漏洞格式，as_list转成list列表，然后fileOperation写入excel。有匹配包装半数，包括独立单词匹配，组件名变体模糊匹配（宽松正则），最后用is_record_relevant集成处理，判断是否属于相关漏洞后用check判断是否通过，后续写入excel。

## parsers：nvd 为例（剩下两个解析器同理）

将nvd数据解析出来， CVE-2024-1234--->OpenSSL 3.0，然后用common中的is_record_relevant判断是否符合匹配，具体如何提取无细究。

## Config_Setting 配置系统文件

在Appconfig里调用getconfig返回的是CONFIG，这个CONFIG是模块加载时创建的唯一CONFIG 
```
CONFIG = _create_default_config()
```
因此mainUI和WebsiteHandle不同文件中更改了调用了config的值的更改如：`UserConfig.set_flag('stop_requested', True)`，一旦改变可以立刻得知。但可能不能多文件同时修改，但好像也遇不到多文件并行修改config的情况。包装了一些修改配置函数。

## template_mapping.json

是一种excel模板，规定每一列填写字段的值。作为 AppConfig.templates可以直接在config中调用，然后输出，即在config中加载为py对象，在FileOperation真正执行写入：

```python
# Config_Setting.py
def _load_mapping_file(mapping_path: Path) -> TemplateMappings:
    with mapping_path.open('r', encoding='utf-8') as mapping_file:
        payload = json.load(mapping_file)

    return TemplateMappings(
        foss=SectionMapping(**payload['foss']),
        oss_vulnerability=SectionMapping(**payload['oss_vulnerability']),
        jira=SectionMapping(**payload['jira']),
    )
```

从JSON文件找到列映射规则，变成代码对象TemplateMappings。

## FileOperation

key：writeVulinfo_batch，包含了写入数据，去重。是否更新取决于时间与来源。

## 完整架构

```
  ┌─────────────────────────────────────────────────────────┐
  │  UI 层                                                  │
  │  main.py ─→ wx.Notebook                                 │
  │    ├── Tab1: UI/mainUI.py       CVE 搜索页               │
  │    └── Tab2: UI/moveOSS2JIRA.py JIRA 转换页              │
  ├─────────────────────────────────────────────────────────┤
  │  调度层                                                  │
  │  WebsIteHandle/WebsiteHandle.py                          │
  │    遍历数据文件 → 调用解析器 → 匹配 → 写入 Excel           │
  ├─────────────────────────────────────────────────────────┤
  │  解析层                                                  │
  │  parsers/common.py        数据结构 + 模糊匹配引擎          │
  │  parsers/nvd_parser.py    NVD JSON → VulnerabilityRecord │
  │  parsers/cnvd_parser.py   CNVD XML → VulnerabilityRecord │
  │  parsers/cnnvd_parser.py  CNNVD → VulnerabilityRecord    │
  ├─────────────────────────────────────────────────────────┤
  │  文件层                                                  │
  │  base/FileOperation.py    Excel 读写、去重索引、JIRA导出   │
  ├─────────────────────────────────────────────────────────┤
  │  配置                                                    │
  │  Config_Setting.py        AppConfig 全局单例              │
  │  template_mapping.json    Excel列号映射                   │
  └─────────────────────────────────────────────────────────┘
```
