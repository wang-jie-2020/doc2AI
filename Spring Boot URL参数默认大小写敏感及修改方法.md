
---
## 😀 默认助手

1. **是，Spring Boot 默认对 URL 查询参数大小写敏感。**  
   例如 `@RequestParam("userId")` 通常不能匹配 `?userid=1` 或 `?USERID=1`。

2. **原因是 Spring 按参数名进行精确匹配**，底层 `HttpServletRequest#getParameter(...)` 以及参数映射的键默认使用区分大小写的字符串。HTTP 本身没有规定查询参数名必须大小写不敏感，具体行为由应用框架决定。

3. **Spring Boot 没有直接配置项可以全局关闭大小写敏感。** 最简单的方式是显式接收不同大小写的参数名：

   ```java
   @GetMapping("/test")
   public String test(
           @RequestParam(value = "userId", required = false) String userId,
           @RequestParam(value = "userid", required = false) String userid) {

       String id = userId != null ? userId : userid;
       return id;
   }
   ```

4. **如果希望全局大小写不敏感，可以注册 `HttpServletRequestWrapper`，统一重写参数读取方法：**

   ```java
   @Component
   public class CaseInsensitiveParamFilter extends OncePerRequestFilter {

       @Override
       protected void doFilterInternal(
               HttpServletRequest request,
               HttpServletResponse response,
               FilterChain filterChain) throws ServletException, IOException {

           filterChain.doFilter(new HttpServletRequestWrapper(request) {
               @Override
               public String getParameter(String name) {
                   String value = super.getParameter(name);
                   if (value != null) {
                       return value;
                   }

                   for (String key : super.getParameterMap().keySet()) {
                       if (key.equalsIgnoreCase(name)) {
                           return super.getParameter(key);
                       }
                   }
                   return null;
               }

               @Override
               public String[] getParameterValues(String name) {
                   String[] values = super.getParameterValues(name);
                   if (values != null) {
                       return values;
                   }

                   for (String key : super.getParameterMap().keySet()) {
                       if (key.equalsIgnoreCase(name)) {
                           return super.getParameterValues(key);
                       }
                   }
                   return null;
               }
           }, response);
       }
   }
   ```

5. 该方案主要适用于 `@RequestParam` 等参数读取场景；如果还要让 `@ModelAttribute`、参数对象绑定也完全大小写不敏感，则需要进一步重写 `getParameterMap()` 或自定义参数绑定逻辑。注意同一请求中如果同时出现 `userId` 和 `USERID`，应先规定冲突处理规则。
