---
{"dg-publish":true,"permalink":"/Program/Mixed/Gradle 依赖管理api,compilyOnly区别/","noteIcon":"","created":"2025-10-16T14:54:26.879+08:00"}
---

## 🧩 一、`compileOnly` 的定义

`compileOnly` 表示：

> **这个依赖只在编译阶段需要，但不会打包进最终的可执行产物（如 jar/war），也不会在运行时提供。**

换句话说：

- ✅ 编译时可见（IDE 自动补全、类型检查都正常）
    
- ❌ 运行时不可见（不会出现在 classpath 中）
    
- ❌ 不会传递给下游模块
    

---

## 🆚 二、与其他依赖方式的对比

| 依赖方式                    | 编译可见    | 运行可见 | 传递给下游模块 | 常见用途                                    |
| ----------------------- | ------- | ---- | ------- | --------------------------------------- |
| **implementation**      | ✅       | ✅    | ❌       | 普通依赖，只在当前模块可见（推荐默认）                     |
| **api**                 | ✅       | ✅    | ✅       | 公共依赖，暴露给依赖此模块的其他模块（常用于 library）         |
| **compileOnly**         | ✅       | ❌    | ❌       | 只在编译期需要，如 Servlet API、Lombok、注解处理器等     |
| **runtimeOnly**         | ❌       | ✅    | ❌       | 只在运行期需要，如 JDBC 驱动、日志实现                  |
| **compileOnlyApi**      | ✅       | ❌    | ✅       | （很少用）与 compileOnly 类似，但会传递编译可见性给下游      |
| **annotationProcessor** | ✅（特殊阶段） | ❌    | ❌       | 仅在编译期注解处理阶段需要，如 MapStruct、Dagger、Lombok |

---

## 🧠 三、举个例子更直观

假设你写了一个 web 应用：

```groovy
dependencies {
    compileOnly 'javax.servlet:javax.servlet-api:4.0.1'
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

### 为什么用 `compileOnly`？

- Spring Boot 的嵌入式 Tomcat 已经包含了 Servlet API；
    
- 你的代码在编译时需要 `HttpServletRequest`、`HttpServletResponse`；
    
- 但运行时由 **Tomcat 提供**，不需要你重复打包。
    

如果你写成 `implementation`：

- 你的 JAR 会重复打包 Servlet API；
    
- 部署到容器时可能冲突（ClassLoader 冲突）。
    

---

## 🧩 四、典型应用场景

|场景|依赖示例|说明|
|---|---|---|
|Web 容器提供类库|`javax.servlet:servlet-api`|在容器（Tomcat、Jetty）里已经有|
|注解辅助编译|`org.projectlombok:lombok`|编译期需要，运行时无意义|
|插件/SDK 开发|`com.google.auto.service:auto-service`|编译时生成 META-INF 文件，运行时由宿主加载|
|SPI 或接口框架|`provided` 类库|由运行时平台提供|

---

## ⚙️ 五、与 Maven 的对比

在 Maven 中类似的 scope 是：

|Gradle|Maven 对应 scope|
|---|---|
|implementation|compile（默认）|
|api|compile（并传递）|
|compileOnly|provided|
|runtimeOnly|runtime|
|annotationProcessor|provided（仅注解处理）|

Gradle 的优点是 **更细粒度、更模块化**，Maven 的 `provided` 在多模块项目中往往语义不够清晰。

---

## 🧩 六、总结要点

|场景|推荐依赖类型|
|---|---|
|普通库依赖|`implementation`|
|要暴露给下游的公共 API 依赖|`api`|
|运行时才需要的库（如数据库驱动）|`runtimeOnly`|
|容器或宿主提供的依赖（Servlet、Lombok）|`compileOnly`|
|编译期注解处理器|`annotationProcessor`|

---

## ✅ 实战建议

1. **在 library 模块中谨慎使用 `api`**  
    不然会导致“依赖泄露”，下游模块看到太多不该看到的类。
    
2. **优先使用 `implementation`**  
    默认隔离编译依赖，构建更快。
    
3. **`compileOnly` 用于“宿主已提供”或“编译辅助”场景**。
    