检索数据库中的条目是很基本常见的功能，实现的方法也很多，常见包括：

1. 基于[Elasticsearch](https://www.elastic.co/cn/elasticsearch) 或 [Lucene](https://lucene.apache.org)这类专业独立的检索引擎实现
2. 基于数据库自带的检索功能实现

虽然基于Elasticsearch这类系统能实现高级灵活的检索功能，但开发和运维成本也将大大增加，
本文将教会你如何利用PostgresSQL内置的功能快速高效的实现大多数中文检索场景。

检索是大多数系统需要的功能，虽然已有很多成熟的检索方案，但多数是面向英文的对中文不友好。
虽然有Elasticsearch这类高级的检索引擎能实现中文检索但其学习和运维成本高，本文将教会你如何使用PostgresSQL数据库自带的功能实现大多数中文检索场景。

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
[pg_trgm](http://www.postgres.cn/docs/11/pgtrgm.html)模块提供了两个字符串相似度计算的函数，
该方法区别于上面两种方法的区别在于利用了概率论的思想来寻找最相似的结果，而不是严格的匹配。

##### Trigram模型介绍
该模块的算法是基于[Trigram](https://en.wikipedia.org/wiki/Trigram)模型实现的，
Trigram全名third grammar，是[N-gram](https://en.wikipedia.org/wiki/N-gram)模型的在N=3时的一个特例。
Trigram的前提思想是假设第X个词的出现只与前面3-1=2个词相关，而与其它任何词都不相关。
在计算相似度时先把一段文字拆分成为多个词，3个一组形成一个Trigram，再找出这个序列中最大相似的Trigram。
以文字`one`Trigram的拆分规则为：

1. 前置两个空格，后置一个空格，变成`  one `
2. 按照从前往后的顺序3个一组拆分为`{  o, on,ne ,one}`

> 你通过通过`SELECT show_trgm('one')`语句来查询如何文本的拆分结果（实际上show_trgm除了调试很少有用）。

> 为什么这里N选择了3而不是其它？是因为N太大会导致计算量成指数上升，而3有着不错的效果同时也能有很好的性能。

#### 使用pg_trgm模块
PostgresSQL默认没有开启pg_trgm模块，需要通过以下语句启用：
```sql
CREATE EXTENSION pg_trgm
```
成功开启后，可以通过`similarity(a, b)`函数判断两句话的相似度，返回的结果是一个`[0,1]`的浮点数，
0表示完全没有一致的字符，1表示完全一样。

如果你想求一句话和一个词的相似度，例如`two words`和`word`的相似度，如果用上面提到的similarity函数会得到`0.36`，
得到这个很低的结果是因为similarity会考虑整体的相似性；
如果想求局部相似性，也就是句子里的`words`和单词`word`相似的，可以使用`word_similarity('word','two words')`得到的结果是`0.8`。

针对以上函数PostgresSQL还提供了简写形式：

- `a % b`：判断`similarity(a, b)`是否大于阀值`pg_trgm.similarity_threshold`（默认0.3）
- `a <% b`：判断`word_similarity(a, b)`是否大于阀值`pg_trgm.strict_word_similarity_threshold`（默认0.6）
- `a <-> b`：a和b之间的整体相似距离，越大表示越不相似，等价于`1-similarity(a, b)`
- `a <<-> b`：a和b之间的局部相似距离，越大表示越不相似，等价于`1-word_similarity(a, b)`

> 你可以通过 `SET pg_trgm.similarity_threshold = 0.8` 语句修改默认的阀值，但通常情况下使用默认值就能获得很好的效果。

### zhparser分词与tsquery


## 如何优化检索性能
在PostgresSQL里提升查询性能最有效地方式是使用索引，针对不同检索方式需要用不同索引来优化，先来看下内置的各种索引和其特点：

### B树（B-tree，Balanced tree）索引
B树索引是使用范围最广的索引，也是执行`CREATE INDEX`时默认使用的索引，几乎所有的数据库都支持B树索引。
B树索引可以有效地用于等值和范围查询，并且也可以用于检索NULL值，排序。

B树索引适用于前匹配的LIKE检索，例如`权力的%`，但不能用于`%权力的`或`%权力的%`，原因在于只有前匹配才能建立B-tree。

### 哈希（Hash）索引
哈希索引原理就像map一样对数据进行KV映射，因此只在等值比较时才有用，但它性能非常好。

### 倒排（GIN，Generalized Inverted Indexes）索引
倒排索引以字或词为关键字进行索引，表中关键字所对应的记录表项记录了出现这个字或词的所有文档，一个表项就是一个字表段，它记录该文档的ID和字符在该文档中出现的位置情况。
倒排索引的结构图如下图：
![倒排索引的结构图](http://p1.meituan.net/scarlett/c1e37cd57d638f3a18a76510c8fb016b17980.png)
由于每个字或词对应的文档数量在动态变化，所以倒排表的建立和维护都较为复杂，但是在查询的时候由于可以一次得到查询关键字所对应的所有文档，所以非常适用用于索引数组值。

### 广义搜索树（GiST，Generalized Search Tree）索引

- PostgresSQL常用索引和适用场景
- 不同检索方式该用什么索引
通用搜索树索引允许你建立普通平衡树结构，也能用于等值和范围比较之外的操作。它们用于索引几何数据类型，也可用于全文搜索。

## 四种检索方法对比与适用场景总结

|  索引类型   | 适用场景  |
|  ----  | ----  |
| B-tree  | 单元格 |
| Hash  | 单元格 |
| GIN  | 单元格 |
| GiST  | 单元格 |
 
