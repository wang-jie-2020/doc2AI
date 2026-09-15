
---
## 😀 默认助手

正常。这是 Spring MVC 的默认行为：

1. `@RequestParam` 获取的是 **request parameters**，来源包括：
   - URL 查询参数：`/submit?username=张三`
   - `application/x-www-form-urlencoded` 请求体
   - `multipart/form-data` 中的普通文本字段
   - multipart 中的文件参数

   ```java
   @PostMapping("/submit")
   public String submit(@RequestParam String username) {
       return username;
   }
   ```

2. 如果同名参数同时出现在 URL 和 Body 中，例如：

   ```text
   /submit?username=url
   username=body
   ```

   `@RequestParam("username")` 看到的是合并后的请求参数，不能依赖它严格区分来源；具体取值可能受参数顺序和容器实现影响。

3. 若要接收普通表单的 **Body 内容**，可以使用：

   ```java
   @PostMapping(
       value = "/submit",
       consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE
   )
   public MultiValueMap<String, String> submit(
           @RequestBody MultiValueMap<String, String> form) {
       return form;
   }
   ```

   这里的 `form` 只解析 `application/x-www-form-urlencoded` 的请求体，不读取 URL 查询参数。

4. 若要接收 multipart 中的 **具体表单部件**，使用 `@RequestPart`：

   ```java
   @PostMapping(
       value = "/upload",
       consumes = MediaType.MULTIPART_FORM_DATA_VALUE
   )
   public String upload(
           @RequestPart("username") String username,
           @RequestPart("file") MultipartFile file) {
       return username + ": " + file.getOriginalFilename();
   }
   ```

5. 因此：
   - 一般表单参数：`@RequestParam`，会合并 URL 和表单内容。
   - 严格读取普通表单 Body：`@RequestBody MultiValueMap<String, String>`。
   - 严格读取 multipart 部件：`@RequestPart`。
   - 文件上传：通常使用 `@RequestPart MultipartFile` 或 `@RequestParam MultipartFile`。
