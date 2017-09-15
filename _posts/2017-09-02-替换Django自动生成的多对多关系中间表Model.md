---
category: Django
---

## 一 起因

使用Django的多对多关系Model非常方便，如两个model，文章和标签，是多对多关系，在Django的models.py可以这么定义：

```python
from django.db import models


class Article(models.Model):
    name = models.CharField(max_length=100)
    tags = models.ManyToManyField('Tag')

    class Meta:
        db_table = 'articles'


class Tag(models.Model):
    name = models.CharField(max_length=100)

    class Meta:
        db_table = 'tags'

```

生成数据库migrations并使用后，如有新的需求，在article上显示加tag的时间，Django自动生成的多对多关系就不能满足了。

此时需要自定义中间表Model，并在`ManyToManyField`选项中设置`through`参数。

但，现有数据库已有存在的数据，如何在新建Model时并使用现有的表。

把新的中间表Model加上:

```python
class Article(models.Model):
    name = models.CharField(max_length=100)
    tags = models.ManyToManyField('Tag', through='ArticleTag')

    class Meta:
        db_table = 'articles'


class ArticleTag(model.Model):
    article = models.ForeignKey('Article')
    tag = models.ForeignKey('Tag')
    created_at = models.DateTimeField(autonow=True)

    class Meta:
        db_table = 'articles_tags'

```

运行`./manage.py makemigrations` 这时会生成一个新的migration文件, 其中operations部分如下所示：

```python

operations = [
    migrations.CreateModel(
        name='AriticleTag',
        fields=[
            ('id', models.AutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
            ('created_at', models.DateTimeField(auto_now=True)),
        ],
        options={
            'db_table': 'articles_tags',
        },
    ),
    migrations.AlterField(
        model_name='article',
        name='tags',
        field=models.ManyToManyField(through='blog.AriticleTag', to='blog.Tag'),
    ),
    migrations.AddField(
        model_name='ariticletag',
        name='article',
        field=models.ForeignKey(on_delete=django.db.models.deletion.CASCADE, to='blog.Article'),
    ),
    migrations.AddField(
        model_name='ariticletag',
        name='tag',
        field=models.ForeignKey(on_delete=django.db.models.deletion.CASCADE, to='blog.Tag'),
    ),
]

```

现在直接运行`./manage.py migrate`会报错，提示articles_tags表已经存在。`django.db.utils.OperationalError: table "articles_tags" already exists`



## 二 解决方案


这种情况，数据库里其实已经有对应的表，不需要实际在数据库建表操作，只需给Django做一个新建Model的声明。

Django有几个Special Operations, 其中的一个是`SeparateDatabaseAndState`：

```
class SeparateDatabaseAndState(database_operations=None, state_operations=None)[source]

A highly specialized operation that let you mix and match the database (schema-changing) and state (autodetector-powering) aspects of operations.

It accepts two list of operations, and when asked to apply state will use the state list, and when asked to apply changes to the database will use the database list. Do not use this operation unless you’re very sure you know what you’re doing.

```
即数据库操作与声明操作可以不一样。 把上面生成的migration文件中creatmodel的操作放到SeparateDatabaseAndStae类的state_operations方法，如下：

```python
operations = [
    migrations.SeparateDatabaseAndState(database_operations=None, state_operations=[
        migrations.CreateModel(
            name='AriticleTag',
            fields=[
                ('id', models.AutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('created_at', models.DateTimeField(auto_now=True)),
            ],
            options={
                'db_table': 'articles_tags',
            },
        ),
        migrations.AlterField(
            model_name='article',
            name='tags',
            field=models.ManyToManyField(through='blog.AriticleTag', to='blog.Tag'),
        ),
        migrations.AddField(
            model_name='articletag',
            name='article',
            field=models.ForeignKey(on_delete=django.db.models.deletion.CASCADE, to='blog.Article'),
        ),
        migrations.AddField(
            model_name='articletag',
            name='tag',
            field=models.ForeignKey(on_delete=django.db.models.deletion.CASCADE, to='blog.Tag'),
        ),
    ]
    ),
]
```

再执行`./manage.py migrate`即可成功，并可以在Django代码中使用新的中间Model`ArticleTag`.
