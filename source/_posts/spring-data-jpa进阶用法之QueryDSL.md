---
title: spring-data-jpa进阶用法之QueryDSL
date: 2019-05-07 23:16:24
categories:
  - jpa
tags:
  - qdsl
  - jpa
---

很多列表查询接口都会有很多复杂的过滤条件。一般都会在controller里面各种拼接条件然后在持久层写好多针对性的查询接口，导致代码可读性差，实现不够优雅。

### QueryDSL
其实QueryDsl可以很优雅的解决上述场景遇到的问题，QueryDSL是一个Java语言编写的通用查询框架。spring-data-jpa对QueryDsl提供了良好的支持。同时spring-data-jpa也针对web做了一些扩展支持。具体可以参考{% link spring-data-jpa的官方文档 https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#core.extensions.querydsl  %}。

### spring-data-jpa的支持

持久层repository继承`QuerydslPredicateExecutor`，即可使用QueryDsl查询。
```java
interface UserRepository extends CrudRepository<User, Long>, QuerydslPredicateExecutor<User> {
}

Predicate predicate = user.firstname.equalsIgnoreCase("dave")
	.and(user.lastname.startsWithIgnoreCase("mathews"));

userRepository.findAll(predicate);
```
web层可以使用 @QuerydslPredicate 标注Predicate。

```java
@Controller
class UserController {

  @Autowired UserRepository repository;

  @RequestMapping(value = "/", method = RequestMethod.GET)
  String index(Model model, @QuerydslPredicate(root = User.class) Predicate predicate,    
          Pageable pageable, @RequestParam MultiValueMap<String, String> parameters) {

    model.addAttribute("users", repository.findAll(predicate, pageable));

    return "index";
  }
}
```
这样包含 ?firstname=Dave&lastname=Matthews的查询将会被`QuerydslPredicateArgumentResolver`解析成 `QUser.user.firstname.eq("Dave").and(QUser.user.lastname.eq("Matthews"))`

有时候我们的参数并不是和实体的属性一一对应，甚至我们需要隐藏一些不可以用来查询的属性。
### 自定义绑定关系
我们可一通过实现 `QuerydslBinderCustomizer`这个接口来自定义参数的绑定关系。
```java
CustomUserQuerydslBinder implements QuerydslBinderCustomizer<QUser> {
    @Override
    public void customize(QuerydslBindings querydslBindings, QUser qUser) {
      //自定义绑定关系
      querydslBindings.excludeUnlistedProperties(true);//使用白名单模式
      querydslBindings.including( //设置属性白名单
            qUser.id,
            qUser.name
      );
      //自定义参数的绑定
       querydslBindings.bind(Expressions.stringPath("sex")).as("type").first((path,value)->
            path.eq(value)
        );
    }
}

```
然后在 `@QuerydslPredicate(bindings=CustomUserQuerydslBinder.class,root=User.class)`中指定。