---
{"dg-publish":true,"permalink":"/Program/Java/What's the difference between @Component, @Repository & @Service annotations in Spring/","noteIcon":"","created":"2024-05-22T16:17:54.146+08:00"}
---

From [Spring Documentation](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-stereotype-annotations):


> The `@Repository` annotation is a marker for any class that fulfils the role or stereotype of a repository (also known as Data Access Object or DAO). Among the uses of this marker is the automatic translation of exceptions, as described in [Exception Translation](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/data-access.html#orm-exception-translation).
>
> Spring provides further [[stereotype\|stereotype]]  annotations: `@Component`, `@Service`, and `@Controller`. `@Component` is a generic [[stereotype\|stereotype]] for any Spring-managed component. `@Repository`, `@Service`, and `@Controller` are specializations of `@Component` for more specific use cases (in the persistence, service, and presentation layers, respectively). Therefore, you can annotate your component classes with `@Component`, but, by annotating them with `@Repository`, `@Service`, or `@Controller` instead, your classes are more properly suited for processing by tools or associating with aspects.
>
> For example, these stereotype annotations make ideal targets for pointcuts. `@Repository`, `@Service`, and `@Controller` can also carry additional semantics in future releases of the Spring Framework. Thus, if you are choosing between using `@Component` or `@Service` for your service layer, `@Service` is clearly the better choice. Similarly, as stated earlier, `@Repository` is already supported as a marker for automatic exception translation in your persistence layer.

| Annotation    | Meaning                                             |
| ------------- | --------------------------------------------------- |
| `@Component`  | generic [[stereotype\|stereotype]] for any Spring-managed component |
| `@Repository` |  [[stereotype\|stereotype]]  for persistence layer                    |
| `@Service`    |  [[stereotype\|stereotype]]  for service layer                        |
| `@Controller` |  [[stereotype\|stereotype]]  for presentation layer (spring-mvc)      |

 [https://stackoverflow.com/questions/6827752/whats-the-difference-between-component-repository-service-annotations-in](https://stackoverflow.com/questions/6827752/whats-the-difference-between-component-repository-service-annotations-in)


以下是上述内容的中文翻译：

---

> `@Repository` 注解是一个标记，用于任何符合仓库角色或存储库角色（也称为数据访问对象或 DAO）的类。此标记的用途之一是自动转换异常，如[异常翻译](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/data-access.html#orm-exception-translation)中所述。
> 
> Spring 还提供了其他的[[Program/Java/Spring框架注解\|Spring框架注解]]：`@Component`、`@Service` 和 `@Controller`。`@Component` 是一个通用的[[Program/Java/Spring框架注解\|Spring框架注解]]，适用于任何由 Spring 管理的组件。`@Repository`、`@Service` 和 `@Controller` 是 `@Component` 的特化版本，用于更具体的使用场景（分别用于持久化层、服务层和表现层）。因此，你可以用 `@Component` 来注解你的组件类，但用 `@Repository`、`@Service` 或 `@Controller` 来注解这些类会更合适，因为这样更适合工具进行处理或与方面（aspects）关联。
> 
> 例如，这些框架注解是为切入点设计的理想目标。`@Repository`、`@Service` 和 `@Controller` 在未来的 Spring Framework 版本中还可能包含更多语义。因此，如果在选择使用 `@Component` 或 `@Service` 来注解你的服务层时，`@Service` 显然是更好的选择。同样，如前所述，`@Repository` 已经被支持为持久化层中的自动异常翻译的标记。

| 注解          | 含义                                               |
| ------------- | --------------------------------------------------- |
| `@Component`  | 任何由 Spring 管理的通用[[Program/Java/Spring框架注解\|Spring框架注解]]               |
| `@Repository` | 持久化层的[[Program/Java/Spring框架注解\|Spring框架注解]]                              |
| `@Service`    | 服务层的[[Program/Java/Spring框架注解\|Spring框架注解]]                                |
| `@Controller` | 表现层（spring-mvc）的[[Program/Java/Spring框架注解\|Spring框架注解]]                  |

参考链接：[https://stackoverflow.com/questions/6827752/whats-the-difference-between-component-repository-service-annotations-in](https://stackoverflow.com/questions/6827752/whats-the-difference-between-component-repository-service-annotations-in)
