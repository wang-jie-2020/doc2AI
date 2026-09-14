
---
## 🤖 Assistant

结论：**JDK 版本和 Maven 版本之间没有绑定关系**，只存在两条单向约束。

Maven 本身只是一个跑在 JVM 上的 Java 程序，它管的是"用什么 JDK 来执行自己"；而项目用哪个 JDK 编译、编译出什么版本的字节码，由 `maven-compiler-plugin` 和 build 配置决定。这两件事是分开的。

## 1. 两个不同层次的 JDK，先分清楚

| 层次 | 由谁决定 | 怎么看 |
|---|---|---|
| Maven 自己运行用的 JDK | Maven 版本的最低要求 + `JAVA_HOME` | `mvn -v` 里的 `Java version` |
| 项目编译/运行的目标版本 | `maven.compiler.release` 或 toolchains | `mvn -v` 看不出来，看 pom |

```bash
$ mvn -v
Apache Maven 3.9.9
Maven home: /opt/apache-maven-3.9.9
Java version: 17.0.11, vendor: Eclipse Adoptium   # ← Maven 运行用的 JDK
```

`mvn` 启动脚本优先用 `JAVA_HOME` 找运行时，找不到才用 PATH 上的 `java`。

## 2. Maven 对 JDK 的最低要求（单向上限，不封顶）

| Maven 版本 | 运行所需最低 JDK |
|---|---|
| Maven 3.0 – 3.8.x | Java 7 |
| Maven 3.9.x | Java 8 |
| Maven 4.0.x | Java 17 |

官方原话是"**最低要求，不设上限**"：Maven 3.9 可以用 JDK 21 跑，只是不能用 JDK 7 跑。

另外还有一层：**插件**也有自己的最低 JDK 和最低 Maven。Maven Plugin API ≥ 3.1.0 要求 Java 8；Maven 社区从 2025 年起建议插件把最低 Maven 提到 3.9.0。所以"我的 Maven 能跑"不代表"我要用的插件能跑"。

## 3. Maven 版本不影响编译目标版本

同一条 Maven 命令、同一份 pom，在不同 JDK 上跑可以产出不同版本的字节码：

```xml
<properties>
  <!-- 用 JDK 17 运行 Maven，产出 Java 8 字节码：完全合法 -->
  <maven.compiler.release>8</maven.compiler.release>
</properties>
```

只要运行 Maven 的 JDK ≥ `release` 指定的版本即可，Maven 3.x 还是 4.x 都一样。

反过来的情况才不合法：**JDK 8 编译不出 Java 17 的字节码**（表现为 `invalid target release: 17`）。

## 4. 两条真正的约束

1. **编译/运行 Maven 的 JDK ≥ 项目目标字节码版本**（否则 `invalid target release`）。
2. **运行 Maven 的 JDK ≥ 项目依赖与插件自身的字节码版本**（否则 `Unsupported class file major version NN`）。

第二条最容易被忽略：依赖 jar 是别人用 JDK 17 编的，你 Maven 跑在 JDK 11 上，就会在解析/执行阶段直接炸。

字节码主版本对照（报错信息里的数字就是这个）：

| Java | class 主版本 |
|---|---|
| 8 | 52 |
| 11 | 55 |
| 17 | 61 |
| 21 | 65 |
| 25 | 69 |

## 5. 一个 Maven 跑多个 JDK：toolchains

需要"编译 JDK"和"Maven 运行 JDK"彻底分开时用 toolchains，它和 Maven 版本无关，只和 pom 里的插件声明有关：

```xml
<!-- ~/.m2/toolchains.xml -->
<toolchains>
  <toolchain>
    <type>jdk</type>
    <provides><version>8</version></provides>
    <configuration><jdkHome>/opt/jdk8</jdkHome></configuration>
  </toolchain>
</toolchains>
```

再在 pom 里启用 `maven-toolchains-plugin` 指定要用的 `<version>` 即可。

## 6. 常见报错对照

| 报错 | 真实原因 |
|---|---|
| `invalid target release: 17` | 运行 Maven 的 JDK 太旧 |
| `Unsupported class file major version 61` | 运行 Maven 的 JDK 比依赖/插件低 |
| `release version 17 not supported` | 同上，Maven 用 JDK 8/11 在编 Spring Boot 3 |
| `mvn: command not found` / 启动即退出 | 没设 `JAVA_HOME`，或 JDK 低于 Maven 最低要求 |

排查顺序：`mvn -v` 看 Maven 用的是哪个 JDK → 看 pom 的 `maven.compiler.release` → 两者对齐。

## 7. 生态里的实际组合（以 Spring Boot 为例）

| 组合 | JDK | Maven |
|---|---|---|
| Spring Boot 2.7 | 8 / 11 / 17 | 3.5+ 即可 |
| Spring Boot 3.x / 4.x | 17+ | **3.6.3+**（官方明确要求） |

所以升级 Spring Boot 3 时，"升 JDK 17"和"升 Maven 到 3.6.3+"通常要一起做；升级 Maven 4 则必须先保证构建环境（含 CI）有 JDK 17。

## 8. 一句话记忆

- **Maven 版本 → 决定"能不能跑起来"的 JDK 下限。**
- **pom 配置 → 决定"编译成哪版 Java"。**
- 两者唯一交点是：跑 Maven 的 JDK 必须 ≥ 目标版本，也必须 ≥ 依赖的字节码版本。
