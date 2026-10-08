
---
## 😀 默认助手

**结论（常见 Spring Cloud + Nacos 场景下，优先级从高到低）：**

1. **代码硬编码 / 代码动态赋值**  
   直接写死的常量、`System.setProperty`、`Environment.getPropertySources().addFirst(...)` 这类方式设置的值最高。

2. **Nacos 远程配置**  
   传统 bootstrap 机制下，Nacos 拉取的远程配置作为高优先级属性源，通常高于命令行参数和本地文件。Nacos 内部一般是：扩展配置/共享配置 < 应用自身配置（还取决于 `extension-configs`、`shared-configs` 声明顺序）。

3. **启动命令行参数**  
   如 `--server.port=8081`，通常高于本地配置文件，但低于 Nacos 远程配置。

4. **`application-{profile}.yml`**  
   高于默认的 `application.yml`。

5. **`application.yml`**

6. **`bootstrap-{profile}.yml` > `bootstrap.yml`**  
   bootstrap 文件本身优先级较低，主要用于提前加载 Nacos 地址、namespace、group 等连接信息。

7. **配置默认值**  
   如 `@Value("${name:default}")`、`@ConfigurationProperties` 字段初始值，仅在外部无对应配置时生效。

**注意事项：**

- 若配置了 `spring.cloud.config.allow-override`、`override-none`、`override-system-properties` 等，可让本地配置/命令行参数覆盖远程配置，优先级随之改变。
- Spring Boot 2.4+ / 新版 Spring Cloud 默认不启用 bootstrap，需引入 `spring-cloud-starter-bootstrap` 或改用 `spring.config.import=nacos:`；两种方式下 Nacos 与本地文件的精确优先级可能不同。
- 实际项目中建议以 `Environment.getPropertySources()` 输出或 `/actuator/env` 端点验证当前生效顺序。
