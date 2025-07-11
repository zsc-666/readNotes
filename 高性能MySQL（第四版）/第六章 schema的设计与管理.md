### 选择数据类型判断要素
更小的通常更好；简单为好；尽量避免存储 NULL

### 选择数据类型步骤
#### 选合适的大类型：数字、字符串、时间
#### 选择具体类型
例如 DATETIME 和 TIMESAMP 列存储相同类型的数据：时间和日期，精确到秒。但是 TIMESTAMP 只使用 DATETIME 一半的存储空间，还会根据时区变化，会自动更新。但是 TIMESTAMP 范围小，1970~2038年

下面只讨论基本类型，其他类型最终都会映射到基本类型，如 INTEGER->INT、BOOL->TINYINT 、NUMERIC -> DECIMAL
##### 整数类型
TINYINT(8)、SMALLINT(16)、MEDIUMINT(24)、INT(32)、BIGINT(64)
存储范围是2(N-1)-1 其中 N 是存储空间的位数。

整数类型可选 UNSIGNED 属性，表述不允许负值 TINYINT UNSIGNED 存储值为 0~255 TINYINT 为 -128~127。

小提示 MYSQL 能为整数类型指定宽度。但是没什么用，INT（1）  和 INT（20）是一样的。

##### 实数类型
浮点类型比DECIMAL 使用更少的空间来存储相同范围的值。FLOAT 使用 4 字节，DOUBLE 8 字节。
由于额外空间需求和计算成本，尽量在对小数精确计算时使用 DECIMAL-例如存储财务数据。
但是大容量场景，可以考虑用BIGINT 替代 DECIMAL。将存储货币单位根据小数位数乘以倍数。假设要存储小数点万分之一分，就把金额同事乘以 100 万，将结果存储在 BIGINT 里。同事避免浮点存储计算不精确和 DECIMAL 精确计算代价高的问题。


#### 字符串类型
##### VARCHAR
VARCHAR 存储可变长度字符串　
使用场景：字符串列最大长度远大于平均长度；列更新很少，所以碎片不是问题；使用了 UTF-8 这样复杂的字符集，每个字符使用了不同的字节数进行存储。

##### CHAR
CHAR 存储固定长度，当存储 CHAR 值时， MYSQL 删除所有尾随空格。需要比较时，用空格填充。
使用场景： CHAR 适合存储非常短的字符串，适用于所有长度都相同的情况。例如用户密码 MD5 值。再比如  设计为保存Ｙ和Ｎ的值，用 char(1) 只要 1 字节，用 varchar(1)需要2字节

##### BINANARY 
存储的是字节 ，填充用的是 \0(零字节)而不是空格，并检索时不会去除填充值。
#### VARBINARY 
存储的是字节

###### 使用注意，慷慨是不明智的
varchar(5) 和 varchar(200) 存储 'hello' 空间开销是一样的。
较大的列会使用更多内存，MySQL 会分配固定大小的内存块来保存值。
最好的策略是只分配真正需要的空间。

##### BLOB 和 TEXT 
BLOB 和 TEXT 专为存储很大的数据而设计的字符串数据类型，分别采用二进制和字符方式存储。 BLOB 没有排序规则和字符集， TEXT 有字符集和排序规则。

MySQL 把每个 BLOB 和 TEXT 当作自己标识的对象处理。存储引擎专门存储它们。当 BLOB 和 TEXT 值太大时，InnoDB 使用独立的 "外部"存储区域，此时值在行内需要 1~4 字节的存储空间，然后在外部存储区域需要足够空间来存储实际的值  

TEXT 与 BLOB 排序是根据 max_sort_length 字节做排序。可以根据需求更改。

TEXT 和 BLOB 不能将完整字符串放到索引，也不能使用索引进行排序

#### 在数据库中存储图像
过去喜欢用 BLOB 存储图像，便于将应用数据保存在一起。但是数据量增长，修改 schema 的操作会因为 BLOB 数据大小越来越慢。
解决办法：将它们写入单独的对象数据存储，用该表来跟踪图像的位置或文件名

#### ENUM
使用 ENUM 代替常规字符串类型。ENUM 存储一组预定义的不同字符串值。MYSQL 存储枚举时会非常紧凑，会根据列表值的数量压缩到  1  或者 2 字节。在内部会将每个值在列表中的位置保存为整数。
![[链接 VARCHAR 和 ENUM列的速度.png]]
由图可见 ENUM join ENUM 很快。
还有一个好处是：将列转换为 ENUM 类型会使表变小约三分之一。

坏处：虽然 ENUM 类型在存储值的方式上非常有效，但更改 ENUM 中有效值会导致需要做 schema 变更。如果没有一个健壮系统来支持自动 schema 变更，那么如果 ENUM 经常更改，会造成很大的不变。

个人看法: 没有实践前还是不要用这个。

#### DATETIME 和 TIMESTAMP
##### DATETIME
从 1000 年到 9999 年，精度为 1 微秒。与时区无关，8 字节存储空间

##### TIMESTAMP
4 字节存储空间，表示 1970 年到 2038 年 1 月 19 日。当插入和更新没有指定 TIMESTAMP 值时，会更新为当前时间。
通过将日期和时间存储为 UNIX 纪元（1970年1月1日以来的秒数u）协调世界时（UTC）的形式，避免 MYSQL 处理的复杂性。使用带符号的 32 位 INT。可表达到 2038 年的时间。使用无符号的 32 位 INT ，可以表达到 2106 年的时间。使用 64 位还可以超出这些范围。
#### BIT
谨慎使用，大多数应用避免使用。存储 true/false 用 TINYINT
#### SET
要存储多个值，可以考虑用 MySQL 的 SET 数据类型。

#### JSON 数据类型
使用 JSON 还是 SQL 列看 JSON 的便捷性是否大于性能。

### 选择标识符
标识符是引用行及通常使用的唯一方式。例如，有一个关于用户的表，希望给用户分配一个数字 ID 或唯一的用户名。可能是主键的部分或全部。

整数通常是标识符的最佳选择，速度快，且能自动递增。 ATUO_INCREMENT 为自动递增的列属性。确保整数不要意外耗尽，导致停机。

ENUM 和 SET 糟糕的情况，尽量避免

字符串类型，如果可能，尽量避免使用字符串类型作为标识符的数据类型，因为消耗空间且比整数慢。
对于完全 "随机" 的字符串要非常小心，MD5()、SHA1()、UUID() 这些都会任意分布在很大的空间，减慢 INSERT 和某些类型 SELECT 查询的速度：
1、插入的值会写到索引的随机位置，使 INSERT 查询变慢，导致页分裂，磁盘随机访问，以及聚簇存储引擎产生聚簇索引碎片。
2、逻辑上相邻的行广泛分布在磁盘和内存中，SELECT 也会变慢
3、对于所有类型查询，随机值会导致缓存性能低下，破坏了局部性。缓存出现大量刷新和不命中。

如果存储通用唯一的标识符（UUID）值，应该删除破折号或使用 UNHEX() 函数将 UUID 值转换为 16 字节的数字，并将其存储在一个BINARY(16),用 HEX() 函数以十六进制格式检索

###  当心自动生成的 schema
自动生成的 schema 可能会导致严重性能问题，需要反复检查确认。缺点：不好扩展，性能容器出问题 优点：开发不用管数据怎么存储的

### MYSQL schema 设计中的陷阱
#### 不能有太多的列
#### 不能有太多的联结 
限制每个联结 61 个表。太多联结对 MYSQL 规划和优化查询是个问题。
#### 全能的枚举
```
CREATE TABEL ... (
	country enum('0','1',...,'31')
)
```
这种应该是一个整数，会被设计为 “字典” 或者 “查找”表的外键。
#### 变相的枚举
只有两个值 Y ， N 就别用 SET 了 用 ENUM。

#### NULL 不是虚拟值
尽量避免使用 NULL。但是有时候该值要表示未知，也需要用。比使用 -1  这种好，-1 使代码复杂化，还容易有 bug。

# 小结
指导原则:尽量避免在设计上出现极端情况，例如，强行执行非常复杂的查询或包含很多列的表设计
- 使用小的，简单的，适当的数据类型，尽量避免使用NULL
- 尝试使用相同的数据类型存储相似或相关的值，尤其是联接条件使用这些值时
- 注意可变长度字符串，可能导致临时表和排序的全长内存分配不乐观
- 尽量用整数作标识符
- 小心使用 ENUM 和 SET。方便但是棘手，最好避免使用BIT


另外 这个文章总结的 数据库设计与优化也值得借鉴
https://juejin.cn/post/7171399220815986695

## 高效的模型设计
很多时候为了尽量提高性能，需要做反范式设计
- 让 Query 尽量减少 Join, 查询频繁但是更新较少的数据，可以考虑 别的表中的字段拿过来在自己身上也存一份数据
- 大字段垂直分拆 - summary 表优化  大字段垂直拆分简单来说就是将自己身上的字段拆分出去放在另外（单独）的表里面。
- 大表水平分拆 - 基于类型的分拆优化。当单台主机无法支撑单个表的访问时，通过这种大表的水平拆分，存放在多台主机的多个数据库来实现整体扩展性的提升。
- 统计表 - 准实时优化。每隔一定时间段进行一次统计后存放在专门设计的统计表中。性能数量级提升且使用户体验上升

### 优化数据类型
优化数据类型提高性能的主要原理在于以下几个方面：
1. 通过选用更“小”的数据类型减少存储空间，使查询相同数据需要的 IO 资源降低；
2. 通过合适的数据类型加速数据的比较；

### 规范的对象命名
公司内部需要统一
注意：
1. 数据库和表名应尽可能和所服务的业务模块名一致； 这样，在 DBA 维护相关数据库对象的时候，新开发人员程序开发过程中，相关技术（或非技术）人员整理业务逻辑和数据关系的时候，都能够非常容易理解其中的关系。
2. 服务于同一子模块的一类表尽量以子模块名（或部分单词）为前缀或后缀； 对同类功能的表增加前缀或者后缀，也是让查看使用该表的各类人员能够很快的根据相关对象的名称就联想到相应的功能，以及相关业务。不论是从维护角度，还是从使用角度来看都会带来非常大的便利性。
3. 表名应尽量包含与所存放数据相对应的单词； 这对于新员工来说尤其重要，要想尽快的熟悉数据，尽快了解相关业务，快速的定位数据库中各表对应的数据意义是非常有帮助的。
4. 字段名称也尽量保持和实际数据相对应 这一点的意义我想各位读者朋友应该都非常的清楚，每个表都会有很多的字段对应数据的各种不同属性，要搞清楚各自代表的含义，除了完整规范的说明文档之外，命名清晰合理的字段名也是一个有用的补充，而且更为直接。
5. 索引名称尽量包含所有的索引键字段名或者缩写，且各字段名在索引名中的顺序应与索引键在索引中的索引顺序一致，且尽量包含一个类似于 idx 或者 ind 之类的前缀或者后缀，以表名其对象类型是索引，同时还可以包含该索引所属表的名称； 这样做最大的好处在于 DBA 在维护过程中能够非常直接清晰的通过索引名称就了解到该索引大部分的信息。
6. 约束等其他对象也应该尽可能包含所属表或其他对象的名称，以表名各自关系。


# 数据库性能不是优化出来的，更多是设计出来的