---
layout: post
title: "如何设计与实现一个分布式索引框架（二）：Schema、API 及倒排索引"
date: 2020-04-22 18:16:55 +0800
comments: true
categories: [index, distrubted, htdadif, recommendation, machine learning]
---

> 这是一个[系列文章](/categories/htdadif/)，大部分内容都来自我过去在小红书发现 Feed 团队工作期间的实践和经验。在介绍的过程中我会尽量不掺杂过多的业务细节，而专注于这背后我个人一些浅薄的设计思想，希望你在阅读完这些文章以后能够直接或者间接地拓展到不同的场景。

在上一篇文章中简单介绍了什么是推荐系统以及实现一个推荐系统的核心组件有哪些，文章最后引入了一个非常重要的概念「索引」，本篇将会首先从框架使用者的角度介绍如何定义索引，框架有哪些 API 可以使用以及从设计者的角度介绍如何实现一个简单的倒排索引。

<!-- more -->

## Schema

在传统的数据库系统中，当我们提到 schema 时通常是指表（table）的逻辑定义，这个定义中会包含这些信息：表名、有哪些列（column）、列名、列的数据类型、主键（primary key）、索引名、索引的列等。非常类似的，在推荐系统中我们也需要这样的信息。框架的使用者需要首先定义好存储的数据实体，如实体名（表名）、实体有哪些字段（列）、字段的名称和数据类型、哪个字段是主键、哪些字段需要创建倒排索引。正如传统数据库系统中通过 SQL 来定义 shcema，我们也需要一种类似的 DDL[^1]。经过一番调研和比较以后，我们选用了 [FlatBuffers](https://google.github.io/flatbuffers) 作为定义 schema 的语言。同 [Protocol Buffers](https://developers.google.com/protocol-buffers)（以下简称 PB）一样，FlatBuffers 也是 Google 开源的一种序列化协议，支持多种主流语言。为什么要选用 FlatBuffers 呢？FlatBuffers 的主页上列举了几个特点，我选取了几个最重要的翻译过来[^2]，如果你熟悉 PB、Thrift 这一类 IDL 应该能很明显看出区别。

- **无需反序列化即可访问序列化后的数据**：将 FlatBuffers 同其它协议区分开来的一个重要原因是 FlatBuffers 通过平展的二进制缓冲区（flat binary buffer）表示层级数据（hierarchical data），因此无需反序列化（parsing/unpacking）即可直接访问数据。同时依然支持数据结构的演变（evolution），保持向前和向后兼容性。
- **高效的内存空间和访问性能**：当访问数据时唯一需要分配的内存就只有数据本身的缓冲区（buffer），不需要任何额外的内存空间（C++ 语言支持，其它语言可能有变化）。FlatBuffers 也非常适合用于 mmap（或者流式处理），允许只有部分缓冲区在内存中。访问序列化后的数据基本等价于访问原始的结构体（struct），只会增加一次额外的跳转（一种虚表）来实现数据格式的演变（evolution）和可选字段。FlatBuffers 旨在应用于那些不接受耗费大量时间和空间访问或者构建序列化数据的项目，例如游戏或者任何其它对性能敏感的应用。点击查看[性能测试](https://google.github.io/flatbuffers/flatbuffers_benchmarks.html)了解更详细的信息。

有兴趣进一步了解设计细节的朋友可以看看官网的 [FlatBuffers Internals](https://google.github.io/flatbuffers/flatbuffers_internals.html) 文档，简单总结就是 FlatBuffers 通过一种特殊的序列化格式（针对更小的内存开销和访问性能设计）相比传统 IDL 更加高性能，同时又兼具传统 IDL 的大部分特性（语言无关、强类型、schema evolution）。当然 FlatBuffers 也不是没有缺点，最明显的一个问题就是为了实现高性能，FlatBuffers 的原始 API 对开发者及其不友好，手动编写序列化或者读取数据[^3]的代码非常容易出错。不过好在这些问题都可以通过自动生成的代码和框架隐藏起来，不需要直接暴露给用户[^4]。前面列举的几个特点为什么对于索引框架如此重要呢？笼统讲当然是为了高性能，不过后面介绍倒排索引的设计时会详细说明一些细节点。

说了这么多还是不知道具体的 schema 长什么样子，下面以一个实际的例子来说明。

```
table NoteInfo {
  note_id:string (id: 0, primary_key);
  ...
  note_gender:NoteGender (id: 29, index_attribute);
  taxonomies:[KeyValueEntry] (id: 30, index_key);
  ...
  breakdown_stats:[BreakdownStats] (id: 47, secondary_key);
}
```

上面是一个完整的索引实体定义，也就是小红书里用户创建的笔记（note）。每一行定义了实体中的字段名称、数据类型以及可选的属性标记。例如 `note_id` 这个字段是笔记的 ID，数据类型是 `string`，`id: 0` 是字段在 FlatBuffers 中的唯一 ID，`primary_key` 表示这个字段是主键。类似的后面列举的几个字段也具有某些特殊含义，例如 `NoteGender` 是一个枚举值，`index_attribute` 表示这是一个索引属性；`[KeyValueEntry]` 是一个 `KeyValueEntry` 类型的数组，`index_key` 表示这是一个倒排索引；`secondary_key` 表示这是一个二级索引。可以看到语法上 FlatBuffers 跟传统 IDL 类似，某种意义上可能还略微简洁一些。定义里有些是 FlatBuffers 官方的语法（如 `id: 0`），还有一些是我们扩展的（如 `primary_key`）[^5]。这里扩展性是非常有必要的，否则这个 IDL 就只能用于序列化而没法作为一种数据的逻辑定义语言来使用了。这些扩展的语法具体是什么意思之后的几篇文章会逐渐展开。

## API

有了 schema 框架就可以理解索引的数据结构了，但是对于使用者来说其实更加关心的是如何「查询」数据。推荐系统的业务特点是一个读远大于写的场景，且在线请求中只会涉及读数据而不涉及写数据，即请求都是只读的。结合上一篇文章的介绍，使用者真正需要用到的 API 基本就是下面几种：

1. 查询正排索引
2. 查询倒排索引
3. 查询二级索引

以 Java 语言为例，实际的 API 大概长这样：

```java
QueryApi.queryByPrimaryKey(Object primaryKey)
QueryApi.queryByIndexKey(String indexKeyName, Object indexKey, long limit, Function<IndexPayload<?>, Boolean> filter)
QueryApi.queryBySecondaryKey(Object primaryKey, String secondaryKeyName, List<SecondaryKey> secondaryKeys)
```

第 1 个 API 通过主键查询正排索引；第 2 个 API 通过倒排索引的字段 key 来查询倒排索引，同时还限定了查询的索引条目数以及一个用户自定义的过滤器；第 3 个 API 通过主键和二级索引 key 查询二级索引。

当然除了以上列举的最基本的 API 以外我们还提供了一些额外的接口，例如为了优化批量查询性能的批量查询接口，为了监控和可视化的索引统计信息查询接口。

## 倒排索引

假设给你一份序列化好的索引数据，要怎么创建倒排索引呢？这里有几个关键的问题需要思考：

1. 如何解析序列化的数据？
2. 如何知道哪些字段需要创建倒排索引？
3. 如何在运行时读取需要创建倒排的字段的值？
4. 倒排索引在内存中的数据结构是什么？
5. 倒排索引的条目列表如何排序？
6. 如何实现在查询倒排索引的同时对条目进行过滤？

第 1 个和第 2 个问题结合前面介绍 schema 时的知识应该很容易解答，只要框架能够提前获取到数据的 schema[^6]，就能对索引数据有一个全局的了解，并能够事先知道哪些字段需要创建倒排索引。

第 3 个问题需要通过 FlatBuffers 提供的[反射 API](https://github.com/google/flatbuffers/blob/master/reflection/reflection.fbs) 来解决[^7]，配合 shcema 就能够从实际的数据中获取某个字段的值。还记得前面没有细讲的一个问题吗？为什么我们选用了 FlatBuffers 作为序列化协议，一个非常重要的原因就是**无需反序列化即可访问序列化后的数据**。在创建倒排索引时这个需求尤其强烈，一个完整的定义有可能包含几十甚至上百个字段，每个字段的大小都是不同的，但是这其中可能只有个位数的字段需要创建倒排索引，如果使用传统的 IDL 反序列化整个对象的时间和空间开销将会非常大，特别是对于有 GC 的语言来说[^8]。因此在这一点上 FlatBuffers 基本完美解决了这个问题。

第 4 个问题思考的角度需要从查询性能出发，既然是索引那必然追求的是查询时间复杂度最小，那就没有比 O(1) 更小的复杂度了。能够实现 O(1) 查找的数据结构最常见的就是 hash map[^9]，在不同语言中这都是非常基础的数据结构，基本不用操心是否需要自己从头开始实现。Hash map 的 key 就是倒排索引 key，value 就是索引的条目列表。而 value 应该用什么数据结构呢？倒排索引的 value 一定是有序的，且通常是倒序排列，最简单的场景用 array 其实就够了，如果需要动态增删那你可能会想到类似 [skip list](https://en.wikipedia.org/wiki/Skip_list) 这样的数据结构。这里有一个细节点需要注意，同一个条目是有可能同时出现在不同的倒排索引中的，因此做好对象复用是节省内存非常关键的点。

回答第 5 个问题前可以先回到介绍 schema 时举的例子，倒排索引的字段是一个特殊的数据结构 `[KeyValueEntry]`，那么这个 `KeyValueEntry` 具体是什么呢？

```
table KeyValueEntry {
  key:string (key);
  value:double;
}
```

这是一个由用户自定义的数据结构，只有两个字段 `key` 和 `value`，前者即是倒排索引 key，而后者即是倒排索引条目的 score，同一个倒排索引 key 下的条目列表将会根据这个 score 从大到小逆序排序。这个特殊的数据结构是框架约定俗成的，只要符合一定条件就可以作为倒排索引的字段类型。

最后一个问题是在推荐系统的业务场景中相当常见的需求，通常查询时会限定查询 top N 的条目，但是对于不同用户这个 top N 可能是不一样的。例如需要过滤掉每个用户历史上曾经有过曝光（impression）的条目，需要根据某些用户画像属性过滤条目等。出于节省内存的原因我们不可能将一个完整定义中的所有字段都直接存放在内存中[^10]，因此限定了只有某些标记了特殊属性的字段才会存储在索引条目中，这也是前面示例中 `index_attribute` 这个标记的作用。因此一个完整的索引条目数据结构大概是这样（以 Java 语言为例）：

```java
public class IndexPayload<T extends Comparable<T>> implements Cloneable {
    private final T primaryKey;
    private final Object indexKey;
    private final double score;
    private final Map<String, Object> attributes;

    public Object getAttribute(String attrName) {
        return attributes.get(attrName);
    }
}
```

上面的 `attributes` 成员变量即是索引属性，key 是标记了索引属性的字段名，value 是对应的值，可以通过 `getAttribute()` 方法查询这个值。前面介绍的 `QueryApi.queryByIndexKey()` 接口中有一个 `filter` 参数，数据类型是 `Function<IndexPayload<?>, Boolean>`，也就是说这个参数是一个函数，输入参数的数据类型是 `IndexPayload`，返回值的数据类型是 `Boolean`。用户需要自己实现过滤器的逻辑，通过 `IndexPayload` 提供的接口来判断是否需要过滤当前条目。

以上就是本篇要介绍的全部内容，简单回顾一下：

- 基于 FlatBuffers 的 schema 定义
- 根据不同索引类型提供不同的查询 API
- 如何在运行时创建倒排索引

下一篇文章依然是围绕索引来介绍，不过重点将会是正排索引，看似一个 hash map 即可解决的问题其实有很多玄机。

[^1]: 为什么不直接用 SQL 呢？首先 SQL 的语法很复杂，很多原语是多余的，这对于使用者来说是不必要的负担。其次我们是实现一个推荐系统而不是一个完备的 DBMS，没必要硬套。最后这个 DDL 需要足够的扩展性来满足针对推荐系统的一些定制需求，关于这一点后面会提到。
[^2]: 需要查看原文和所有特点的朋友请转到 FlatBuffers 的官网
[^3]: 这里我刻意没有用「反序列化」这个词，理论上 FlatBuffers 是没有反序列化这个概念的，buffer is data（缓冲区即数据）。
[^4]: 框架使用者甚至不需要知道底层用的是 FlatBuffers
[^5]: 同样的设计思想在 [FoundationDB Record Layer](https://github.com/FoundationDB/fdb-record-layer/blob/master/docs/Overview.md) 里也有所体现，只不过它使用的是 PB 作为 DDL，相比之下 FlatBuffers 的语法会更加简洁。
[^6]: 实际在实现时是通过框架暴露的注册 schema 的 API 由用户来提供这些信息
[^7]: 截止 2020 年 4 月 FlatBuffers 官方依然没有提供 Java 语言的反射 API，有需要的朋友可以参考 [#4019](https://github.com/google/flatbuffers/pull/4019) 这个 PR，虽然这个 PR 也烂尾了。
[^8]: 如果你使用的是 Java 语言，即使用对象池这个问题也是没法优化的，类似 PB 这样的协议对于对象池的支持可以说是相当不友好。
[^9]: 这里暂时忽略掉哈希碰撞
[^10]: 至于完整的数据存放在哪里后续的文章中会介绍
