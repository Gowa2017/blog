---
title: Django中的Models
categories:
  - Python
date: 2020-02-06 22:17:41
updated: 2020-02-06 22:17:41
tags: 
  - Python
  - Django
  - Evennia
  - Mud
---
在 [Django][Django] 中有一个 ORM （对象关系模型，Object Relation Model）系统，在 Python 对象与数据库表间进行映射。对于这个 Model 是什么我们就需要来看一下了。根据[官方定义][Models]，每个 **模型** 模型是有关数据的唯一确定的信息源，它包含了我们需要存储的数据的重要的字段及行为。通常，每个 **模型** 映射到数据库中的一张表。

<!--more-->

基本上：

- 每个 **模型** 都是一个 Python 类，其继承自 [django.db.models.Model][ModelsSrc]。
- **模型** 的每个属性都代表了数据库表中的字段
- 有了这些，Django为您提供了一个自动生成的数据库访问API。使我们能进行 [数据库查询][API]

也就是说，我们用 Python 的类来定义我们的 **模型**，就能通过这个模型创建数据，丢到数据库去了。

# 例子
比如，我们建立一个 *Person* 模型：

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
```
我们定义了两个字段， *first_name, last_name*。

我们执行命令：

```sh
 python manage.py makemigrations polls
 Migrations for 'polls':
  polls/migrations/0001_initial.py
    - Create model Person
```

这个命令会检测我们对文件的修改，并边修改的部分存储为一次 **迁移**，由于我们是第一次修改，所以其有 initial 字样。

我们执行命令：

```sh
python manage.py sqlmigrate polls 0001
```

这可以让我们得出 SQL 语句的输出：

```sql
BEGIN;
--
-- Create model Person
--
CREATE TABLE "polls_person" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "first_name" varchar(30) NOT NULL, "last_name" varchar(30) NOT NULL);
COMMIT;
```

如果我们为模型再增加一个 **middle_name** 字段，我们可以来对比一下前后的内容：

```py
class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
    middle_name = models.CharField(max_length=30,default='')
```
```sh
python manage.py makemigrations polls
Migrations for 'polls':
  polls/migrations/0002_person_middle_name.py
    - Add field middle_name to person
```
```sh
python manage.py sqlmigrate polls 0002
```

```sql
BEGIN;
--
-- Add field middle_name to person
--
CREATE TABLE "new__polls_person" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "middle_name" varchar(30) NOT NULL, "first_name" varchar(30) NOT NULL, "last_name" varchar(30) NOT NULL);
INSERT INTO "new__polls_person" ("id", "first_name", "last_name", "middle_name") SELECT "id", "first_name", "last_name", '' FROM "polls_person";
DROP TABLE "polls_person";
ALTER TABLE "new__polls_person" RENAME TO "polls_person";
COMMIT;
```
这将会把原来的表删除，然后把新表的数据插入。

最后，我们把这个模型变更应用到数据库：

```sh
python manage.py migrate
Operations to perform:
  Apply all migrations: polls
Running migrations:
  Applying polls.0001_initial... OK
  Applying polls.0002_person_middle_name... OK
```
将会逐个应用我们的 **迁移** 文件。
# 使用

现在我们已经建立了一个模型  **Person**，现在我们来使用他。先进入我们的 django shell

```sh
python manage.py shell
```

我们之前已经说过，一个模型就是一张表，所以我们可以在这个模型上进行查询（对表进行查询，修改等操作）.

```py
>>> from polls.models import Person
>>> Person.objects.all() # 暂无对象
<QuerySet []>
>>> p = Person(first_name='gowa',last_name='json') # 创建对象
>>> p.save() # 存储到数据库
>>> p.id # 查看ID
1
>>> Person.objects.all() # 已经有对象
<QuerySet [<Person: Person object (1)>]>
```
# 关联关系

先理解一下几个概念，两个表，A，B。

- 一对多：A 表中的某个 值被 B 表中的多个记录引用。[`ForeignKey`](https://docs.djangoproject.com/zh-hans/3.0/ref/models/fields/#django.db.models.ForeignKey)
- 多对一：B 表中的多个记录引用 A 表中的某个值。[`ForeignKey`](https://docs.djangoproject.com/zh-hans/3.0/ref/models/fields/#django.db.models.ForeignKey)
- 多对多：A 表中的某个值可能被 B 中的多个记录引用，B 表中的某个值也可能被 A 表中的多个值引用。[`ManyToManyField`](https://docs.djangoproject.com/zh-hans/3.0/ref/models/fields/#django.db.models.ManyToManyField)
- 一对一：[`OneToOneField`](https://docs.djangoproject.com/zh-hans/3.0/ref/models/fields/#django.db.models.OneToOneField) 

## 多对多模型

义一个多对多的关联关系，使用 [`django.db.models.ManyToManyField`](https://docs.djangoproject.com/zh-hans/3.0/ref/models/fields/#django.db.models.ManyToManyField) 类。就和其他 [`Field`](https://docs.djangoproject.com/zh-hans/3.0/ref/models/fields/#django.db.models.Field) 字段类型一样，只需要在你模型中添加一个值为该类的属性。

就比如，官方的例子，在做 Pizza 的时候，会有多种配料 Topping，某种 Pizza 会有多种配料，但是多种 Pizza 会用到同一种配料。

```python
class Items(models.Model):
    name = models.CharField(max_length=255)


class Char(models.Model):
    name = models.CharField(max_length=255)
    items = models.ManyToManyField(Items)

```



我们可以执行命令来看一下发生了什么：



```sh

python manage.py sqlmigrate polls 0002
BEGIN;
--
-- Create model Items
--
CREATE TABLE `polls_items` (`id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY, `name` varchar(255) NOT NULL);
--
-- Create model Char
--
CREATE TABLE `polls_char` (`id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY, `name` varchar(255) NOT NULL);
CREATE TABLE `polls_char_items` (`id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY, `char_id` integer NOT NULL, `items_id` integer NOT NULL);
ALTER TABLE `polls_char_items` ADD CONSTRAINT `polls_char_items_char_id_0fb458ce_fk_polls_char_id` FOREIGN KEY (`char_id`) REFERENCES `polls_char` (`id`);
ALTER TABLE `polls_char_items` ADD CONSTRAINT `polls_char_items_items_id_dcb7790e_fk_polls_items_id` FOREIGN KEY (`items_id`) REFERENCES `polls_items` (`id`);
ALTER TABLE `polls_char_items` ADD CONSTRAINT `polls_char_items_char_id_items_id_fe06818e_uniq` UNIQUE (`char_id`, `items_id`);
COMMIT;
```

可以看到，自动生成了一个中间表  `polls_char_items`，然后通过在中间设置关联关系来进行实现。

而对于关系的更新就比较的简单了：

```python
from polls.models import Char, Items

Char.objects.create(name="张三")
Char.objects.create(name="李四")
Char.objects.create(name="王五")
Items.objects.create(name="刀")
Items.objects.create(name="枪")
Items.objects.create(name="剑")
Items.objects.create(name="戟")

char = Char.objects.get(name='张三')
char.items.add(1) # 直接添加索引
item = Items(name='牛剑')
item.save() # 直接添加对象
char.items.add(item)

char.items.remove(item) 
char.items.remove(1)
char.items.clear() # 清除所有关联关系
```





[Django]: https://www.djangoproject.com/
[Models]: https://docs.djangoproject.com/en/3.0/topics/db/models/
[ModelsSrc]: https://docs.djangoproject.com/en/3.0/ref/models/instances/#django.db.models.Model
[API]: https://docs.djangoproject.com/en/3.0/topics/db/queries/
