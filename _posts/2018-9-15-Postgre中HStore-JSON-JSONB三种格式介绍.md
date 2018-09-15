在Postgresql数据库中，添加了几种字段类型来满足在sql中使用类似与NoSql的键值对进行存取。

# [HStore](https://www.postgresql.org/docs/9.3/static/hstore.html)

Hstore是按**键值对**的形式直接存储在数据库中。

它的优点是不需要提前定义键，直接插入要保存的内容即可，且可以在键上加索引，缺点是不支持嵌套对象存储。

例子：

``` sql
CREATE TABLE hstore_test (
  id serial PRIMARY KEY,
  attributes hstore
);
```

可以将任何内容的键值对插入到`attributes`列中

``` sql
INSERT INTO hstore_test (attributes) VALUES (
 'author    => "luli",
  date  => "2018"'
);
```

然后可以根据不同的键或值进行查询

``` sql
select attributes->'author' as author 
from hstore_test 
where attributes->'date' = '2018'; 
```

这里明显可以看出hstore足够的灵活性，特别是再加上postgrel独有的GIN，GIST索引。

# [JSON](https://www.postgresql.org/docs/9.6/static/datatype-json.html)

JSON是直接的**文本存储**，所以可以嵌套对象，但是**不可以使用索引**。

json中可以存储有效的[JSON](http://json.org/)值，例如``null`, `true`, `[1,false,"string",{"foo":"bar"}]`, `{"foo":"bar","baz":[null]}``这些都可以存入json中。

``` sql
SELECT '{"foo": [true, "bar"], "tags": {"a": 1, "b": null}}'::json;
```

下面是对JSON格式操作的一些例子：

``` sql
--获取JSON对象
select '{"a": {"b":"foo"}}'::json->'a';

--获取JSON对象的文本（as text）
select '{"a":1,"b":2}'::json->>'b'；

--根据路径获取JSON对象
select '{"a": {"b":{"c": "foo"}}}'::json#>'{a,b}'；

--根据路径获取JSON对象的文本
select '{"a":[1,2,3],"b":[4,5,6]}'::json#>>'{a,2}'
```

再补充一个实际中的例子：

``` sql
CREATE TABLE json_test (
  id serial PRIMARY KEY,
  attributes json
);

INSERT INTO json_test (attributes) 
VALUES ( '{
    "hello": "hello-value",
    "world": {
        "world_type": "earth",
        "world_value": "world"
    }
}');

--这里的俩条查询语句我之所以用 #>>这个操作符，因为等号右边是文本
select * from json_test
where attributes #>> '{hello}' = 'hello-value';

select * from json_test 
where attributes #>> '{world,world_type}' = 'earth';
```

除了上述的->，->>， #>， #>>这些操作符外，还有很多操作符以及函数，可以说是相当丰富了。详细信息请移步[官网JSON函数和操作符](https://www.postgresql.org/docs/current/static/functions-json.html)，这里就不再赘述。

# [JSONB](https://www.postgresql.org/docs/9.6/static/datatype-json.html)

站在用户的角度来说JSONB和JSON的操作方式几乎相同，都接受相同值作为输入。它们二者之间主要的区别是效率。

首先，JSON 数据类型存储输入文本的精确拷贝，处理函数必须在每个执行上重新解析，而JSONB数据以分解的二进制格式存储，这使得它由于添加了转换机制而在输入上稍微慢些，但是在处理上明显更快，因为不需要重新解析。（JSONB格式要比JSON格式多占用空间)

其次，JSONB支持索引，这亦是一个明显的优势。

# 结论

那么在面对这三种格式该如何选择呢？

1. 如果你的需求是__键值对存取(无对象嵌套)__，并有__大量操作__，且需要__建立索引__，那么就选择HStore。

2. 如果你的需求只是**文本存取**，并**无**大量操作，那么就选择JSON，JSON存储速度快。

3. 如果你的需求是__文本存取__，且有**大量操作**，并__需要建立索引__，那么就选择JSONB。







