---
theme: z-blue
highlight: docco
---
# 前言
事务是数据库中对处理的逻辑分组，每个组或事务可以包含一个或多个操作，比如跨多个文档的读写操作。

MongoDB 支持跨多个操作、集合、数据库、文档和分片的 ACID 事务。

本章介绍事务并会定义 ACID 对于数据库的意义，重点介绍如何在应用程序中使用它们，并会针对 MongoDB 事务调优提供一些技巧。

本章主要内容如下：
- 什么是事务；
- 如何使用事务；
- 对应用程序的事务限制进行调优。

# 事务简介

事务是数据库中处理的`逻辑单元`，包括一个或多个数据库操作，既可以是读操作，也可以是写操作。

事务的一个重要方面是它永远`不会只完成一部分`——它要么成功，要么失败。

## ACID的定义

ACID 是一个“真正”事务所需要具备的一组属性集合。ACID 是原子性（atomicity）、一致性（consistency）、隔离性（isolation）和持久性（durability）的缩写。

`原子性`确保了事务中的所有操作要么都被应用，要么都不被应用。事务永远不能应用部分操作，要么被提交，要么被中止。

`一致性`确保了如果事务成功，那么数据库将从一个一致性状态转移到下一个一致性状态。

`隔离性`是允许多个事务同时在数据库中运行的属性。它保证了一个事务不会查看到任何其他事务的部分结果，这意味着多个事务并行运行与依次运行每个事务所获得的结果相同。

`持久性`确保了在提交事务时，即使系统发生故障，所有数据也都会保持持久化。

当数据库满足所有这些属性并且只有`成功的事务`才会`被处理`时，它就被称为是`符合 ACID 的数据库`。如果在事务完成之前发生故障，那么 ACID 确保不会更改任何数据。

MongoDB 是一个分布式数据库，它支持跨副本集和/或分片的 ACID 事务。

## 如何使用事务

MongoDB 提供了两种 API 来使用事务。
- 第一种是与关系数据库类似的语法（如start_transaction 和 commit_transaction），称为核心 API；
- 第二种称为回调 API，这是使用事务的推荐方法。

核心 API 不为大多数错误提供重试逻辑，它要求开发人员为操作、事务提交函数以及所需的任何重试和错误逻辑手动编写代码。

回调 API 提供了一个单独的函数，该函数封装了大量功能，包括启动与指定逻辑会话关联的事务、执行作为回调函数提供的函数以及提交事务（或在出现错误时中止）。此函数还包含了处理提交错误的重试逻辑。

node 核心 API 实例代码：
```JavaScript
   const { mongoose } = this.ctx.app;
    const session = await this.ctx.getSession();
    const db = mongoose.connection;
    try {
      let data = { name : 'ceshi' };
      const Cat = new this.ctx.model.Cat();
      for (let key in data) {
        Cat[key] = data[key]
      }
      await db
        .collection('cats')
        .insertOne(Cat, { session });
      // 提交事务
      await session.commitTransaction();
      return 'ok';
    } catch (err) {
      // 回滚事务
      const res = await session.abortTransaction();
      console.log(res)
      this.ctx.logger.error(new Error(err));
    } finally {
      await session.endSession();
    }
    // 执行后,数据库中多了一条 { name: 'ceshi'} 的记录
```

# 对应用程序的事务限制进行调优

## 时间和oplog大小限制

在 MongoDB 事务中有两类主要的限制。第一类与事务的时间限制有关，控制特定事务可以运行多长时间、事务等待获取锁的时间以及所有事务将运行的最大长度。第二类与 MongoDB 的 oplog 条目和单个条目的大小限制有关。

### 时间限制

事务的默认最大运行时间是 1 分钟。可以通过在 mongod 实例级别上修改 transactionLifetimeLimitSeconds 的限制来增加。
对于分片集群，必须在`所有分片`副本集成员上设置该参数。超过此时间后，事务将被视为已过期，并由定期运行的清理进程中止。

### oplog大小限制

MongoDB 会创建出与事务中写操作数量相同的 oplog 条目。但是，每个 oplog 条目必须在 16MB 的 BSON 文档大小限制之内。