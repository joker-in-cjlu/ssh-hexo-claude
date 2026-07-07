---
title: Internship Experiences
date: 2026-07-07
tags: [安全测试, DevOps]
---

## 三种安全测试方法对比

| | SCA | SAST | DAST |
|------|------|------|------|
| 全称 | Software Composition Analysis | Static Application Security Testing | Dynamic Application Security Testing |
| 测什么 | **第三方依赖** | **自己的源代码** | **运行中的应用** |
| 什么时候 | 开发/构建阶段 | 编码/提交阶段 | 测试/上线后 |
| 像什么 | 查原材料合格证 | 看图审图纸 | 造完了试车 |

## 举个例子

你的项目引用了 `log4j 2.14.0`：

- **SCA** → 发现 log4j 2.14.0 有 CVE-2021-44228 漏洞，建议升级到 2.17.0

你的代码写了：

```java
String query = "SELECT * FROM users WHERE name = '" + userName + "'";
```

- **SAST** → 发现第 42 行有 SQL 注入风险，建议用 PreparedStatement

应用部署到测试环境后：

- **DAST** → 发送恶意 payload 后发现 `/api/login` 返回了异常堆栈信息

## 如何实现

### SCA

扫描依赖清单（`package.json`、`pom.xml`、`build.gradle` 等），对比 CVE/NVD 漏洞数据库，报告已知漏洞并建议升级版本。

常用工具：**Snyk**、**OWASP Dependency-Check**、**Black Duck**

### SAST

解析源代码为 AST（抽象语法树），匹配危险模式，跟踪数据流，找出 SQL 注入、XSS、硬编码密码等问题。

常用工具：**SonarQube**、**Checkmarx**、**Semgrep**

### DAST

爬取运行中的网站，自动注入恶意 payload，根据响应判断是否存在漏洞。

常用工具：**OWASP ZAP**、**Burp Suite**、**Acunetix**

## 一句话总结

```
SCA  = 你用的第三方库有没有已知漏洞
SAST = 你写的代码有没有安全缺陷（白盒，看源码）
DAST = 你跑起来的应用能不能被攻破（黑盒，看行为）

三者互补，不是替代关系。
```
