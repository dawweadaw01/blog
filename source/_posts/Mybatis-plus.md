---
title: Mybatis-plus 使用技巧
top: true
date: 2023-7-3
cover: https://s2.loli.net/2023/07/16/9wqUGTRdE6v1IZF.jpg
coverWidth: 1200
coverHeight: 750
tag:
   - Mybatis-plus
categories: 
   - Mybatis-plus
---







# Mybatis-plus

# 实现批量插入

在实际的项目开发过程中，常常遇到批量保存数据的场景，当数据量比较少，比如只有几条数据的情况下，我们可以使用 *for* 循环来 *insert* 数据，但如果数据量比较多的情况下就不行，特别是并发的情况下，因为这样会增加数据库的负担。

我们通过查看 *mybatis-plus* 源码发现，*mybatis-plus* 的 *IService* API 接口提供了批量插入的接口：

```java
public interface IService<T> {
    ......
    /**
     * 插入（批量）
     *
     * @param entityList 实体对象集合
     */
    @Transactional(rollbackFor = Exception.class)
    default boolean saveBatch(Collection<T> entityList) {
        return saveBatch(entityList, DEFAULT_BATCH_SIZE);
    }

```

查看该批量插入的实现方法源码

```java
public boolean saveBatch(Collection<T> entityList, int batchSize) {
    String sqlStatement = sqlStatement(SqlMethod.INSERT_ONE);
    try (SqlSession batchSqlSession = sqlSessionBatch()) {
        int i = 0;
        // 在for循环中循环调用insert
        for (T anEntityList : entityList) {
            batchSqlSession.insert(sqlStatement, anEntityList);
            if (i >= 1 && i % batchSize == 0) {
                batchSqlSession.flushStatements();
            }
            i++;
        }
        batchSqlSession.flushStatements();
    }
    return true;
}

```

从源码可以看到，所谓的批量插入就是一个 for 循环插入，很明显，这不是我们想要结果。而当我们继续阅读 mybatis-plus 的源码可以发现，在 com.baomidou.mybatisplus.extension.injector.methods.InsertBatchSomeColumn 包中已经为我们实现了真正意义上的批量插入方法，这里就不贴实现的源码了，有兴趣的可以去看看。

因此，我们需要做的就是生效该批量了插入方法，从而可以让我们通过 Mapper 来调用它。

****实现批量插入****

****引入依赖****

```xml
<!-- mybatis plus 与 springboot 整合的依赖 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.0</version>
</dependency>

<!-- mybatis plus extension 包含了 mybatis plus core -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-extension</artifactId>
    <version>3.4.0</version>
</dependency>
```

****编写自定义SQL注入类****

```java
public class OwnerSqlInjector extends DefaultSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass);
        methodList.add(new InsertBatchSomeColumn());
        return methodList;
    }
}
```

****将该类注入到 Bean 中****

```java
package cn.test.mybatisplus.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisPlusConfig {
    @Bean
    public OwnerSqlInjector ownerSqlInjector() {
        return new OwnerSqlInjector();
    }
}
```

****将批量插入方法扩展进 BaseMapper 中****

```java
@Mapper
public interface OwnerMapper<T> extends BaseMapper<T> {
    Integer insertBatchSomeColumn(Collection<T> entityList);
}
```

****测试验证****

![Untitled](images/Mybatis-plus/Untitled.png)

可以看到sql语句为批量插入的sql语句