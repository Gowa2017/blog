---
title: SQLAlchemy统一进行数据库连接操作
categories:
  - Python
date: 2021-02-05 08:13:55
updated: 2021-02-05 08:13:55
tags: 
  - Python
  - SQLAlchemy
---

SQLAlchemy SQL 工具集和 ORM 是 Python 与数据库进行交互的综合性工具。它有几个不同的区域和功能，可以单独使用，也可以结合起来使用。它的主要组件按如下进行分层组织。

<!--more-->

# 架构

![](https://docs.sqlalchemy.org/en/13/_images/sqla_arch_small.png)

上图中，最重要的两部分就是 ORM 和 SQL Expression Language。SQL Expression 可以独立于 ORM 使用。

SQLAlchemy  的官方文档也分成了三部分：

- ORM。介绍了对象关系映射。如果我们想要使用自动构建的高级 API，就看这部分文档。
- CORE。介绍了 SQLAlchemy SQL 和数据库集成，核心就是 SQL Expression Language，它可以独立使用来构建可操纵的 SQL 表达式（如修改执行返回等）。和 ORM 的以域为中心模式不同，它使用了以 Schema 为中心的使用模式。
- Dialects。支持的数据库和 DBAPI 后端。

我暂时不做编程开发，只做数据查询修改相关的，所以看看第二部分。

# SQL Expression Language

SQLAlchemy Expression Language 展示了一个表现关系数据库结构和使用 Python 构建的表达式系统。 Python 构建被模仿来尽可能的接近底层数据库，但在不同数据库后端的实现上提供了一些抽象。尽管这个构建尝试在不同的后端间用一致的结构来表示相同的概念，但不能隐藏在不同后端间的特定概念。因此 Expression Language 展示了一个撰写后端中立的 SQL 表达式，但并不强制要表达式后端中立的。

Expression Language 和 ORM 形成了对照，ORM 是构建在 Expression Language 上的一个 API。ORM 是一个高级和抽象的使用模式，也是一个 Expression Language 的实际应用。

在 Expression Language 和 ORM 的使用模式上会有重叠的地方，第一眼看起来可能会觉得非常相似。ORM 通过 *用户dkyid域MODEl* 来组织数据的内容，这个 model 透明的持久化并被底层的存储 MOdel更新。Expression Language 从字面的 schem 和 SQL 表达式来组织数据的内容。

## 版本检查

```python
import sqlalchemy
sqlalchemy.__version__ 
```

## 连接

这里我们只使用 SQLite 内存库。这会简化我们的操作。要连接我们使用  `create_engine`

```python
from sqlalchemy import create_engine
engine = create_engine('sqlite:///:memory:', echo=True)
```

*echo* 标志是设置 SQLALchemy 日志（通过 Python 的标准 *logging* 模块实现）的快速方式。开了这个，我们可以看到所有的产生的 SQL。

`create_engine` 返回的是一个 *Engine* 实例，代表了到数据库的核心接口，通过操纵数据库细节和DBAPI的 *dialect* 来适配。在这个例子中，SQLite dialect 将会解释到 Python 内建的 *sqlite3* 模块。

当第一次执行类似 `Engine.execute, Engine.connect` 这样的方法时，*Engine* 才会真正建立与数据库的 DBAPI 连接。

## 定义和建立表

SQL Expression Language 大多数时候通过表的列来构建表达式。在 SQLALchemy 中，列常通过对象 *Column* 来表示，而 *Column* 与 *Table* 相关联。一系列的 *Table* 与他们相关的子对象被叫做 **数据库 metadata**。这个例子中我们就用到了几个表对象，但实际上 SA 可以导入一个数据库的所有表（叫做表反射）。

我们在一个 *Metadata* 内定义我们的表，使用 *Table* 来构建，这将会生成常规的 `CREATE TABLE` 语句。

```python
from sqlalchemy import Table, Column, Integer, String, MetaData, ForeignKey
metadata = MetaData()
users = Table('users', metadata,
     Column('id', Integer, primary_key=True),
     Column('name', String),
     Column('fullname', String),
 )

addresses = Table('addresses', metadata,
   Column('id', Integer, primary_key=True),
   Column('user_id', None, ForeignKey('users.id')),
   Column('email_address', String, nullable=False)
  )
```

选择就是告诉 *Metadata* 我们需要建立表了。

```python
metadata.create_all(engine)
```

输出:

```
2021-02-05 08:55:17,545 INFO sqlalchemy.engine.base.Engine SELECT CAST('test plain returns' AS VARCHAR(60)) AS anon_1
2021-02-05 08:55:17,545 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,545 INFO sqlalchemy.engine.base.Engine SELECT CAST('test unicode returns' AS VARCHAR(60)) AS anon_1
2021-02-05 08:55:17,545 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,546 INFO sqlalchemy.engine.base.Engine PRAGMA main.table_info("users")
2021-02-05 08:55:17,546 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,546 INFO sqlalchemy.engine.base.Engine PRAGMA temp.table_info("users")
2021-02-05 08:55:17,547 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,547 INFO sqlalchemy.engine.base.Engine PRAGMA main.table_info("addresses")
2021-02-05 08:55:17,547 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,547 INFO sqlalchemy.engine.base.Engine PRAGMA temp.table_info("addresses")
2021-02-05 08:55:17,547 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,548 INFO sqlalchemy.engine.base.Engine
CREATE TABLE users (
	id INTEGER NOT NULL,
	name VARCHAR,
	fullname VARCHAR,
	PRIMARY KEY (id)
)


2021-02-05 08:55:17,548 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,548 INFO sqlalchemy.engine.base.Engine COMMIT
2021-02-05 08:55:17,548 INFO sqlalchemy.engine.base.Engine
CREATE TABLE addresses (
	id INTEGER NOT NULL,
	user_id INTEGER,
	email_address VARCHAR NOT NULL,
	PRIMARY KEY (id),
	FOREIGN KEY(user_id) REFERENCES users (id)
)


2021-02-05 08:55:17,549 INFO sqlalchemy.engine.base.Engine ()
2021-02-05 08:55:17,549 INFO sqlalchemy.engine.base.Engine COMMIT
```

## 插入

```python
ins = users.insert()
str(ins)
```

```
'INSERT INTO users (id, name, fullname) VALUES (:id, :name, :fullname)'
```

我们可以插入指定字段：

```python
ins = users.insert().values(name='jack', fullname='Jack Jones')
str(ins)
'INSERT INTO users (name, fullname) VALUES (:name, :fullname)'
```

上面 `valus` 方法限制了 VALUES 语句到两个列上，我们实际放入的值并没有渲染到字符串中，我们得到的是绑定变量。这就证明，我们的数据是存储在 *Insert* 构建中，但典型的是在语句真正执行后才会打印出来；因为数据是由字面值组成，SQLALchemy 自动的对他们生成了绑定变量。我们可以就一下编译后的格式来查看数据。

```python
{'name': 'jack', 'fullname': 'Jack Jones'}
```

## 执行

*Insert* 的有意义部分在于执行它。在这个例子中，我们关注于显式执行一个 SQL 构建的方法，后面会关注一些更简单的方式。*engine* 对象是一个数据库连接资源并能向数据库提交 SQL。使用 *Engine.connect* 来获取一个数据库连接：

```python
conn = engine.connect()
conn
<sqlalchemy.engine.base.Connection object at 0x10e067fd0>
```

```python
result = conn.execute(ins)
2021-02-05 09:03:44,526 INFO sqlalchemy.engine.base.Engine INSERT INTO users (name, fullname) VALUES (?, ?)
2021-02-05 09:03:44,526 INFO sqlalchemy.engine.base.Engine ('jack', 'Jack Jones')
2021-02-05 09:03:44,526 INFO sqlalchemy.engine.base.Engine COMMIT
```

现在 INSERT 语句提交到数据库了。但我们先得到的是 `?` 而不是命名的绑定变量。为何？因为在执行时，*Connection* 使用 SQLite 的 **dialect** 来帮助生成语句；而当我们使用 `str()` 的时候，这个语句并不知道 dialect ，因此回退到使用默认的命名参数。我们可以通过手动来查看：

```python
ins.bind = engine
str(ins)
'INSERT INTO users (name, fullname) VALUES (?, ?)'
```

那么 *result* 又是什么样的呢？*Connection* 对象引用一个 DBAPI 连接，这个结果，*ResultProxy*，类似于 DBAPI 游标对象。在 INSERT 的情况下，我们可以获取重要的信息比如主键：

```python
result.inserted_primary_key
[1]

```

主键 1 是 SQLite 自动生成的，但这只是我们没有指定 `id` 列在 *Insert* 语句的情况下。SQLALchemy 总是知道如何获取一个新生成的主键值，即使在不同的数据库间获取的方式并不相同；因为每个数据库的 *Dialect* 知道获取正确值的特定步骤（注意 *ReslultProxy.inserted_primary_key* 返回的是一个列表，这可以支持组合的主键）。这些方法，从使用`cursor.lastrowid` 到选择一个数据库特定的函数，到使用 `INSERT .. RETURING` 语法，这些都是透明的。

## 执行多个语句

上面的例子对于显示 expression language 构建的多种行为有点不够。通常，*Insert* 会与参数一起编译后送到 *Connection* 的 *execute* 方法，这里就没必要使用 *values* 关键词了。现在我们建立一个通用的 *Insert* 语句，并进行通常的使用方式：

```python
ins = users.insert()
conn.execute(ins, id=2, name='wendy', fullname='Wendy Williams')
2021-02-05 09:15:52,805 INFO sqlalchemy.engine.base.Engine INSERT INTO users (id, name, fullname) VALUES (?, ?, ?)
2021-02-05 09:15:52,806 INFO sqlalchemy.engine.base.Engine (2, 'wendy', 'Wendy Williams')
2021-02-05 09:15:52,806 INFO sqlalchemy.engine.base.Engine COMMIT
<sqlalchemy.engine.result.ResultProxy object at 0x10e08e710>
```

要执行多个插入的话，我们使用 DBAPI 的 `executemany` 方法，我们可以发送一个字典列表，每个字典包含了一系列需要插入的参数：

```python
conn.execute(addresses.insert(), [
   {'user_id': 1, 'email_address' : 'jack@yahoo.com'},
   {'user_id': 1, 'email_address' : 'jack@msn.com'},
   {'user_id': 2, 'email_address' : 'www@www.org'},
   {'user_id': 2, 'email_address' : 'wendy@aol.com'},
])
2021-02-05 09:17:32,799 INFO sqlalchemy.engine.base.Engine INSERT INTO addresses (user_id, email_address) VALUES (?, ?)
2021-02-05 09:17:32,799 INFO sqlalchemy.engine.base.Engine ((1, 'jack@yahoo.com'), (1, 'jack@msn.com'), (2, 'www@www.org'), (2, 'wendy@aol.com'))
2021-02-05 09:17:32,799 INFO sqlalchemy.engine.base.Engine COMMIT
<sqlalchemy.engine.result.ResultProxy object at 0x10e08ebd0>
```

当执行多个参数集时，每个字典必须有相同的键集合。这是因为，*Insert* 语句会和第一个字段进行编译，并假设所有接下来的参数字典都与此语句兼容。

类似 `executemany` 风格的调用对每个 `insert(), update(), delete()` 都可用。

## 选择

```python
from sqlalchemy.sql import select
s = select([users])
result = conn.execute(s)
2021-02-05 09:20:06,092 INFO sqlalchemy.engine.base.Engine SELECT users.id, users.name, users.fullname
FROM users
2021-02-05 09:20:06,092 INFO sqlalchemy.engine.base.Engine ()
```

上面例子中，我们使用了 `select()`，不过我们将 `users` 表放到了列参数位置，并进行执行。SQLALchemy 会将表扩展为它的列集合，然后生成一个 FROM 语句。返回的结果再次是一个 *ResultProxy* 对象，类似 DBAPI 的游标，包含了如 `fetchone(), fetchall()` 等dhfa。这些方法返回 行对象，通过 *RowProxy* 类提供。 *result* 对象可以直接遍历：

```python
for row in result:
  print(row)
(1, 'jack', 'Jack Jones')
(2, 'wendy', 'Wendy Williams')
```

我们看到，每个 *RowProxy* 产生一个类似元组的结果。*RowProxy* 行为就像一个 Python 字典和元组的混合，但包含了几个获取每个列的方法。一个方法就是使用 Python 的字典访问语法，使用列表：

```python
result = conn.execute(s)
row = result.fetchone()
print("name:", row['name'], "; fullname:", row['fullname'])
name: jack ; fullname: Jack Jones
```

再就是使用 Python 的序列方式，整型索引：

```python
row = result.fetchone()
print("name:", row[1], "; fullname:", row[2])
name: wendy ; fullname: Wendy Williams
```

一个更特殊的列访问方法是使用直接对应到一个特定列的 SQL 构建作为字典键；

```python
for row in conn.execute(s):
    print("name:", row[users.c.name], "; fullname:", row[users.c.fullname])
2021-02-05 09:28:28,761 INFO sqlalchemy.engine.base.Engine SELECT users.id, users.name, users.fullname
FROM users
2021-02-05 09:28:28,761 INFO sqlalchemy.engine.base.Engine ()
name: jack ; fullname: Jack Jones
name: wendy ; fullname: Wendy Williams
```

*ResultProxy* 的自动关闭行为 会在所有挂起的结果行被取走后关闭 DBAPI 的 *cursor*  对象。如果在还有数据时我们想要关闭，可以手动：

```python
result.close()
```

### 特定行选择

```python
s = select([users.c.name, users.c.fullname])
result = conn.execute(s)
for row in result:
     print(row)
    
2021-02-05 09:32:20,303 INFO sqlalchemy.engine.base.Engine SELECT users.name, users.fullname
FROM users
2021-02-05 09:32:20,303 INFO sqlalchemy.engine.base.Engine ()    
```

我们观察一下 FROM。生成的语句中包含两个唯一的区域，SELECT 的列 和 FROM 的表，我们的 `select` 只有一个列的序列。这是如何工作的。我们放两个表到 `select` 中去：

```python
for row in conn.execute(select([users, addresses])):
    print(row)
    
(1, 'jack', 'Jack Jones', 1, 1, 'jack@yahoo.com')
(1, 'jack', 'Jack Jones', 2, 1, 'jack@msn.com')
(1, 'jack', 'Jack Jones', 3, 2, 'www@www.org')
(1, 'jack', 'Jack Jones', 4, 2, 'wendy@aol.com')
(2, 'wendy', 'Wendy Williams', 1, 1, 'jack@yahoo.com')
(2, 'wendy', 'Wendy Williams', 2, 1, 'jack@msn.com')
(2, 'wendy', 'Wendy Williams', 3, 2, 'www@www.org')
(2, 'wendy', 'Wendy Williams', 4, 2, 'wendy@aol.com')    
```

这将两个表都放到了 FROM 中。但是，也混乱了起来。这和 SQL 的 join 相似，做了一个笛卡尔积。因此，我们需要一个 WHERE。我们通过 `Select.where` 来实现：

```python
s = select([users, addresses]).where(users.c.id == addresses.c.user_id)
for row in conn.execute(s):
     print(row)
```

`users.c.id == addresses.c.user_id` 只是简单使用了 Python 的 `==` 操作符在列上，但这不是应该返回  True 或者  False 么？

```python
users.c.id == addresses.c.user_id
<sqlalchemy.sql.elements.BinaryExpression object at 0x10e08e290>
str(users.c.id == addresses.c.user_id)
'users.id = addresses.user_id'
```

这是因为 `__eq__()`  的重载了。



## 操作符



# 表映射

SQLAlchemy 历史上有两种风格的表映射。原来的映射 API 通常被叫做 '传统风格'，现在更自动的被叫做 '声明式' 风格。SQLAlchemy 把它们叫做 **命令式映射** 和 **声明式映射**。

两种风格可以交换使用，且结果相同：一个用户定义的类，它拥有一个 Mapper 配置到一个可选择的单元，通常，用一个 `Table` 对象表示。

声明和命令式的映射都以一个 ORM 的 `registry` 对象开始，它维护了一系列被映射的类。所有的映射都会存在这个 `registry`。

## 声明式

当前，声明式是最典型的用法。最常用的形式是：首先用 `declarative_base()` 函数来构造一个基类，这将会从其衍生的所有子类应用声明映射过程。

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base

# declarative base class
Base = declarative_base()

# an example mapping using the base
class User(Base):
    __tablename__ = 'user'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    fullname = Column(String)
    nickname = Column(String)
```

基类引用了一个维护一系列相关映射类的 `registry` 对象。`declarative_base()` 实际上可以看成是首先构造 `registry`，然后调用生成基类方法：

```python
from sqlalchemy.orm import registry

# equivalent to Base = declarative_base()

mapper_registry = registry()
Base = mapper_registry.generate_base()
```

直接应用 `registry` 来访问不同使用场景下的多种映射形式。

- [Declarative Mapping using a Decorator (no declarative base)](#orm-declarative-decorator) - declarative mapping using a decorator, rather than a base class.
- [Imperative (a.k.a. Classical) Mappings](#orm-imperative-mapping) - imperative mapping, specifying all mapping arguments directly rather than scanning a class.

## 反射表

一个简单的使用将一个从数据库反射出来的表到一个类的方法是使用混合映射，将 `Table.autoload_with` 传递到 `Table`。

```python
engine = create_engine("postgresql://user:pass@hostname/my_existing_database")

class MyClass(Base):
    __table__ = Table(
        'mytable',
        Base.metadata,
        autoload_with=engine
    )
```

上面这个例子的劣势在于，它期待一个活跃的数据库连接。

### 延迟反射

为了解决上面的情况，一个叫 `DeferredReflection` 的扩展可用，这将推迟到一个显示的 `DeferredReflection.prepare()` 被调用的时候再执行映射。

```python
from sqlalchemy.orm import declarative_base
from sqlalchemy.ext.declarative import DeferredReflection

Base = declarative_base()

class Reflected(DeferredReflection):
    __abstract__ = True

class Foo(Reflected, Base):
    __tablename__ = 'foo'
    bars = relationship("Bar")

class Bar(Reflected, Base):
    __tablename__ = 'bar'

    foo_id = Column(Integer, ForeignKey('foo.id'))
engine = create_engine("postgresql://user:pass@hostname/my_existing_database")
Reflected.prepare(engine)
```



## 自动映射

为 `sqlalchemy.ext.declarative` 定义了一个扩展来自动的生成映射类和关系表。

### 基本应用

最简单的用法是将数据库反射到一个新的 Model，我们使用  `automap_base()` 来生成一个基类，然后调用  `AutomapBase.prepare()` 来让它进行反射被产生映射表：

```python
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

Base = automap_base()

# engine, suppose it has two tables 'user' and 'address' set up
engine = create_engine("sqlite:///mydatabase.db")

# reflect the tables
Base.prepare(engine, reflect=True)

# mapped classes are now created with names by default
# matching that of the table name.
User = Base.classes.user
Address = Base.classes.address

session = Session(engine)

# rudimentary relationships are produced
session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
session.commit()

# collection-based relationships are by default named
# "<classname>_collection"
print (u1.address_collection)
```

### 从 MetaData 生成映射

我们可以传递一个预定义的 `MetaData` 对象到 `automap_base()`。这个对象可以用多种方式来构造：通过代码、序列化文件或者是从使用  `MetaData.reflect()` 得来的结果。下面我们会展示一下反射和显式表声明：

```python
from sqlalchemy import create_engine, MetaData, Table, Column, ForeignKey
from sqlalchemy.ext.automap import automap_base
engine = create_engine("sqlite:///mydatabase.db")

# produce our own MetaData object
metadata = MetaData()

# we can reflect it ourselves from a database, using options
# such as 'only' to limit what tables we look at...
metadata.reflect(engine, only=['user', 'address'])

# ... or just define our own Table objects with it (or combine both)
Table('user_order', metadata,
                Column('id', Integer, primary_key=True),
                Column('user_id', ForeignKey('user.id'))
            )

# we can then produce a set of mappings from this MetaData.
Base = automap_base(metadata=metadata)

# calling prepare() just sets up mapped classes and relationships.
Base.prepare()

# mapped classes are ready
User, Address, Order = Base.classes.user, Base.classes.address,\
    Base.classes.user_order
```

