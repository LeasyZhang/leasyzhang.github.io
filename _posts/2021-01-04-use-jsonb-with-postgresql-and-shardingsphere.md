---
layout:     post
title:      "Use PostgreSQL jsonb type with ShardingSphere"
subtitle:   "Within Spring Data Jpa"
date:       2021-01-04
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - ShardingSphere
    - PostgreSQL
    - Spring
---

> "在Shardingsphere框架中使用Postgres Jsonb数据"

这篇文章介绍如何在ShardingSphere框架中使用postgres jsonb类型的数据：

经过本地验证，ShardingSphere无法支持JPA和PostgreSQL Jsonb同时使用，如果同时使用会报

```java
Can't infer the SQL type to use for an instance of com.fasterxml.jackson.databind.node.ObjectNode. Use setObject() with an explicit Types value to specify the type to use.
```

这种错误，为了支持Jsonb数据类型，只能使用SQL语句来操作包含Jsonb的表。

本文操作的对象是:

```java
public class News implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    @Column(name = "title")
    private String title;

    @Column(name = "author")
    private String author;

    @Column(name = "content", columnDefinition = "jsonb")
    private String content;
}
```

这是Jpa定义的Repo类

```java
@Repository
public interface NewsRepo extends JpaRepository<News, Long> {
}
```

这个类如果不扩展的话是不支持对news表的增删改查的，我们继承一个接口来扩展这个Repo:

```java
public interface NewsRepoCustom {
    News saveOne(News news);
}
```

然后让NewsRepo继承这个接口。
接下来我们需要自己实现扩展出来的接口，在Jpa框架下如果要执行自定义SQL，需要注入EntityManager这个对象。

```java
@Component
public class NewsRepoImpl implements NewsRepoCustom {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public News saveOne(final News news) {
        return null;
    }
}
```

PostgreSQL中向数据表插入jsonb数据，需要使用jsonb函数，如果向本文使用的news表插入数据，SQL应该是：

```sql
insert into news (title, author, content) values ('', '', ''::jsonb);
```

所以我们需要在NewsRepoImpl的saveOne方法中执行Raw SQL：

```java
Session session = entityManager.unwrap(Session.class);
return session.doReturningWork(connection -> {
    String sql = "insert into news(title, author, content) values (?, ?, ?::jsonb)";
    PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
    int paramIndex = 1;
    ps.setString(paramIndex++, news.getTitle());
    ps.setString(paramIndex++, news.getAuthor());
    ps.setString(paramIndex, news.getContent());
    ps.executeUpdate();
    ResultSet result = ps.getGeneratedKeys();
    while (result.next()) {
        news.setId(result.getLong(1));
    }
    ps.close();
    return news;
});
```

这样子就可以实现向数据表中插入jsonb类型的数据了。如果想要查看完整的例子，可以参考这个[Github Repo](https://github.com/LeasyZhang/spring-boot-example/tree/master/spring-boot-shardingsphere-jsonb)。
