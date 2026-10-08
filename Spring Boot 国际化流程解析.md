
---
## 😀 默认助手





**结论：** LocaleChangeInterceptor 的作用是**修改** Locale（通过 URL 参数切换），而 Locale 的**读取/解析**由 `LocaleResolver` 自身在请求处理流程中被框架直接调用，**不依赖任何拦截器**。

澄清两点职责：

1. **LocaleChangeInterceptor ≠ Locale 解析器**：前者是"写入/切换"（默认不注册，需要手动通过 `WebMvcConfigurer.addInterceptors` 添加才能生效），后者是"读取当前 Locale"。
2. **Locale 解析的实际调用方**：在每次请求处理时，以下位置会直接调用 `LocaleResolver.resolveLocale(request)`，并不经过拦截器链：
   - `DispatcherServlet.buildLocaleContext()`（请求入口）
   - `AbstractHandlerMapping.getHandler()`（匹配 Controller 时）
   - `View` 渲染阶段、@ResponseBody 消息转换阶段
   - `RequestContextHolder` / `LocaleContextHolder` 中持有的 ThreadLocal

> 补充：Spring Boot 的 `WebMvcAutoConfiguration` 会自动注册 `AcceptHeaderLocaleResolver`（默认按 `Accept-Language` 请求头解析），所以只要 LocaleResolver Bean 在容器中，框架就会在处理请求时自动调用它完成解析，与拦截器无关。LocaleChangeInterceptor 只是额外提供了"运行时切换语言"的能力，属于锦上添花。
