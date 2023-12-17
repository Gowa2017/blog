---
title: pandas类似于SQL一样的使用操作
categories:
  - Python
date: 2018-12-04 20:37:46
updated: 2018-12-04 20:37:46
tags: 
  - Python
  - Pandas
---
主要是企业为了对接数据，结构并不一致，感觉使用 SQL 实现起来是有点类的。有点为难 MySQL 了。还是准备打算用 Python 来使用。
<!--more-->

# Python 连接 MySQL

我们可以使用包 pymysql。

先安装 pymysql:

```sh
pip install -U pymysql
```

在 py 脚本内使用

```py
import pymysql
import pandas as pd
pymysql.install_as_MySQLdb()
import MySQLdb
```

执行 `pymysql.install_as_MySQLdb()` 后，使用 MySQLdb 的地方会不知不觉的使用  pymysql 了。

将我们 MySQL 的配置写在一个字典内：

```py
dbconf = {
    'host': 'sunny-catalyst-130304.asia-east2.mysql
',
    'user': 'google',
    'password': 'fajdlbudadf',
    'db': 'game',
    'port': 3306
}

db = MySQLdb.connect(**dbconf)

```

这样我们就可以连接我们的 MySQL 了。连接后返回的其实是一个 pymysql.connections 类的实例。我们就可以用他其中的很多方法了。

其实现在的模块好像都有这样做法，或者说类。就是在实例内会有字段保存我们获取的结果等，通过方法来获取这样。

# Connection 实例使用。

## query()
本来我是看到返回的 db 有 query 方法的，我们也可以进行使用。但是又看到这个方面上的注释：

```py
    # The following methods are INTERNAL USE ONLY (called from Cursor)
    def query(self, sql, unbuffered=False):
        # if DEBUG:
        #     print("DEBUG: sending query:", sql)
        if isinstance(sql, text_type) and not (JYTHON or IRONPYTHON):
            if PY2:
                sql = sql.encode(self.encoding)
            else:
                sql = sql.encode(self.encoding, 'surrogateescape')
        self._execute_command(COMMAND.COM_QUERY, sql)
        self._affected_rows = self._read_query_result(unbuffered=unbuffered)
        return self._affected_rows
```

嗷，这个方法是给 Cursor 来使用的。一般我们都是得到一个 Cursor 实例来执行查询的。

## cursor()

```py
    def cursor(self, cursor=None):
        """
        Create a new cursor to execute queries with.

        :param cursor: The type of cursor to create; one of :py:class:`Cursor`,
            :py:class:`SSCursor`, :py:class:`DictCursor`, or :py:class:`SSDictCursor`.
            None means use Cursor.
        """
        if cursor:
            return cursor(self)
        return self.cursorclass(self)

```

然后才是我们调用 cursor 的方法来进行查询的：

```py
cur = db.cursor()
cur.execute('select sysdate() from dual')
cur.fetchone()
```

注意：调用来 `fetchone(), fetchall(), fetchmany()` 这些方法后结果就会减少哦。

# pd.read_sql(sql, db, index_col=)

pandas 就更给力了，可以直接从 sql 进行查询获取得到结果的哦。

```py
df = pd.read_sql('select * from information_schema.TABLES',db)
print(df)
```

这样我们就可以随便的像处理其他数据一样处理了。

# 列拆分

我们构造一个一个字段逗号分隔存储多个值的情况：

```py
df = pd.DataFrame({'id':['A','B','C'],'item':['1,2,3','3,4,5,6,7','101,102,103']})
df = df.set_index('id')

id	item
A	1,2,3
B	3,4,5,6,7
C	101,102,103
```

我们把 A 列当做字符串按 `,` 拆分的话，得到的是一个数组，我们可以提供一个额外参数 *expand=True* 来拆分结果也变成列：

```py
df.item.str.split(',', expand=True)



id	0	1	2	3	4					
A	1	2	3	None	None
B	3	4	5	6	7
C	101	102	103	None	None



```

# pd.stack() 函数

这个函数，将列堆叠到指定索引级别。

返回一个重新整形后的，具有一个多级索引的 帧 或 列，与当前的帧相比较，多级索引的内层索引级别最高。这通过旋转最当前帧的列来建立最内层的级别。

* 如果当前列只有一个级别，那么输出就是一个系列
* 如果当前列有多个级别，那么就通过我们指定的级别来转换。

## 参数

* **level** *int, str, list, default -1* 从列轴堆到索引轴的级别。定义为一个索引或者标签，或者一个标签列表等。
* **dropna**

继续上面的例子：

```py
df.item.str.split(',',expand=True).stack()
id   
A   0      1
    1      2
    2      3
B   0      3
    1      4
    2      5
    3      6
    4      7
C   0    101
    1    102
    2    103
dtype: object
```

这样的情况下，我们就是一个具有两级索引的系列了，但我们不需要那个二级索引，所以要丢掉：

```py
df.item.str.split(',',expand=True).stack().reset_index(level=1, drop=True)
id
A      1
A      2
A      3
B      3
B      4
B      5
B      6
B      7
C    101
C    102
C    103
dtype: object
```
`pd.reset_index(level= , drop=True)` 的意思是把对应级别的索引丢弃，drop=True的意思，是不要把丢弃的索引给插为新列。

结果是一个系列，我们可以把列命名一下。

`series.reset_index(level=None, drop=False, inplace=False)` 方法会重新生成一个帧，把以前的索引变成一列。

level, drop 参数的意思，同上一个reset_index()方法。

