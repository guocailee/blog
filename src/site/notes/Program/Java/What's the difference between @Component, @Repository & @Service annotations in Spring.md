---
{"dg-publish":true,"permalink":"/Program/Java/What's the difference between @Component, @Repository & @Service annotations in Spring/","noteIcon":""}
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
