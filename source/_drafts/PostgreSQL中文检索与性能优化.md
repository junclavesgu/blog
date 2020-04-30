检索数据库中的条目是大多数系统都需要的功能，实现的方法也很多，诸如：

1. 基于[Elasticsearch](https://www.elastic.co/cn/elasticsearch) 或 [Lucene](https://lucene.apache.org)这类专业独立的检索引擎实现
2. 基于数据库自带的检索功能实现

虽然基于Elasticsearch这类系统能实现高级灵活的检索功能，但开发和运维成本也将大大增加，
本文将教会你如何利用PostgresSQL内置的功能快速高效的实现大多数中文检索场景。

## 实现中文检索的四种方式

### 使用LIKE通配符
LIKE语句通过通配符实现文字检索，例如`SELECT * FROM movies WHERE title LIKE '权力的%'`语句能找出所有名称以`权力的`为开头的电影。

LIKE语句支持两种通配符：

- `%`：代表任意个数的字符
- `_`：代表一个字符

例如以下匹配结果：
```
'abc' LIKE 'abc'    true
'abc' LIKE 'a%'     true
'abc' LIKE '_b_'    true
'abc' LIKE 'abc_'   false
```

如果你想忽略大小写可以通过`ILIKE`实现，例如 `'abc' LIKE 'aBc'`会返回true。

PostgresSQL还提供了LIKE语句的一些简写形式：

- `~~` 等价于 `LIKE`
- `~~*` 等价于 `ILIKE`
- `!~~` 等价于 `NOT LIKE`
- `!~~*` 等价于 `NOT ILIKE`

### 使用SIMILAR TO正则表达式
通过SIMILAR语句能让我们借助正则表达式实现更高级的匹配，而不是像LIKE那样简单的通配符，例如以下语句：
```
'abc' SIMILAR TO 'abc'      true
'abc' SIMILAR TO 'a'        false
'abc' SIMILAR TO '%(b|d)%'  true
'abc' SIMILAR TO '(b|c)%'   false
```

有了正则表达式，还可以通过内置的`substring`函数提取出特定的字符串：
```
substring('foobar' from '%#"o_b#"%' for '#')   oob
substring('foobar' from '#"o_b#"%' for '#')    NULL
```

PostgresSQL同样提供了SIMILAR TO语句的简写形式：

- `~ 'abc'` 等价于 `SIMILAR TO '.*abc.*'`，以及对应的取反操作`!~ 'abc'`
- `~* 'abc'` 等价于 `SIMILAR TO '.*abc.*'` 但会忽略大小写，以及对应的取反操作`!~* 'abc'`

> 这些SQL语法都是PostgresSQL特有的，虽然便捷但不推荐使用，因为兼容性和可读性不好。

### pg_trgm 字符串相似度

### zhparser分词与tsquery


## 如何优化检索性能
- PostgresSQL常用索引和适用场景
- 不同检索方式该用什么索引

## 三种方法对比与适用场景总结
 
