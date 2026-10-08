
---
## 😀 默认助手

在 Java 项目中，`mvnw` 和 `mvnw.cmd` 是 **Maven Wrapper** 的启动脚本，用来在项目中固定并自动使用指定版本的 Maven。

简单说：  
即使你的电脑没有安装 Maven，也可以通过它们执行 Maven 命令。

---

## 1. `mvnw` 是什么？

`mvnw` 是给 Linux / macOS / Unix 系统使用的脚本。

例如：

```bash
./mvnw clean package
```

等价于使用 Maven 执行：

```bash
mvn clean package
```

但区别是：`./mvnw` 会使用项目指定的 Maven 版本。

---

## 2. `mvnw.cmd` 是什么？

`mvnw.cmd` 是给 Windows 系统使用的脚本。

例如：

```cmd
mvnw.cmd clean package
```

或者在 PowerShell 中：

```powershell
.\mvnw.cmd clean package
```

---

## 3. 它们的作用

### 1）不需要本地提前安装 Maven

如果项目里有 Maven Wrapper，你可以直接运行：

```bash
./mvnw spring-boot:run
```

或者 Windows 下：

```cmd
mvnw.cmd spring-boot:run
```

它会自动下载所需版本的 Maven。

---

### 2）保证团队 Maven 版本一致

项目中通常会有这个文件：

```text
.mvn/wrapper/maven-wrapper.properties
```

里面会指定 Maven 版本，例如：

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
```

这表示项目使用 Maven `3.9.6`。

这样团队成员、CI/CD 环境、服务器都可以使用同一个 Maven 版本，避免因为 Maven 版本不同导致构建结果不一致。

---

### 3）方便 CI/CD 构建

在 Jenkins、GitHub Actions、GitLab CI 等环境里，可以直接执行：

```bash
./mvnw clean package
```

不需要提前配置 Maven 环境。

---

## 4. 常见项目结构

一般 Maven Wrapper 会包含这些文件：

```text
mvnw
mvnw.cmd
.mvn/
  wrapper/
    maven-wrapper.properties
    maven-wrapper.jar
```

有些新版项目里可能没有 `maven-wrapper.jar`，而是使用其他方式下载 wrapper。

---

## 5. 和 `mvn` 的区别

| 命令 | 说明 |
|---|---|
| `mvn` | 使用你电脑全局安装的 Maven |
| `./mvnw` | 使用项目指定版本的 Maven |
| `mvnw.cmd` | Windows 下使用项目指定版本的 Maven |

---

## 6. 常用命令示例

Linux/macOS：

```bash
./mvnw clean
./mvnw clean package
./mvnw test
./mvnw spring-boot:run
```

Windows：

```cmd
mvnw.cmd clean
mvnw.cmd clean package
mvnw.cmd test
mvnw.cmd spring-boot:run
```

---

## 7. 如果提示没有权限

在 Linux/macOS 下可能需要加执行权限：

```bash
chmod +x mvnw
```

然后再执行：

```bash
./mvnw clean package
```

---

总结：  
`mvnw` 和 `mvnw.cmd` 是 Maven Wrapper 脚本，用来让项目自动使用指定版本的 Maven，避免本地未安装 Maven 或版本不一致的问题。`mvnw` 用于 Linux/macOS，`mvnw.cmd` 用于 Windows。
