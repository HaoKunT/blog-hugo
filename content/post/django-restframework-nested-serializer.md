+++
title = "Django Restframework 嵌套序列化"  # 文章标题
date = 2019-08-04T23:29:01+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["django"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["后端","Python","技术"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

`Django`是一款重量级`Python`后端框架，有许多插件与之集成，其中`Django Restframework`是非常成熟的后台接口生成模块。本文围绕嵌套序列化的问题进行研究

## 一个例子
假设一个简单的业务逻辑，我们有一个用户表，一个邮箱表，其中用户表的主键是邮箱表的外键，`Django`的模型为
``` python
class User(models.Model):
    # 这里尽量不要使用Auto，不要把所有东西都交给Django来做
    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=128, null=True, blank=True)
    objects = models.Manager()

class Email(models.Model):
    email = models.EmailField()
    user = models.ForeignKey(User, on_delete=models.CASCADE, null=True, blank=True)
    objects = models.Manager()
```
这里外键为`user_id`，非常简单的模型

### Serializer
序列化，这里序列化起到什么作用我就不展开细讲了，网上教程很多
``` python
class EmailSerializers(ModelSerializer):
    class Meta:
        model = Email

class UserSerializers(ModelSerializer):
    # 这里增加一个Serilizer
    email = EmailSerializers(many=True)
    class Meta:
        model = User
```
大家可以很清楚的看到，在`UserSerializers`中我增加了一个`EmailSerializers`字段，这就是嵌套序列化(*Nested Serilizer*)

> 我自己起的名字哈哈哈

### 嵌套的用处
嵌套主要是为了解决反向查找的问题的，相信有一种需求就是在查用户的时候顺带将他所拥有的所有邮箱查找出来，如果不使用嵌套，可能你需要自己重写查询函数，比较麻烦

### ViewSet
``` python
class UserModelViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializers

class EmailModelViewSet(ModelViewSet):
    queryset = Email.objects.all()
    serializer_class = EmailSerializers
```
一切看起来都很美好

是的，看起来都很美好，

**错！那是建立在没有写任务的基础上！！！**

## 如果出现写任务的时候呢？
那就直接gg了

> 这里的写任务指的是`update`和`create`，对应Rest里面就是`Post`,`PUT`,`PATCH`

好了吐槽完了，我们来看一下官方是怎么看这个问题的

在源码上有这样一段，这是我在报错了之后找到的

``` python
# ModelSerializer & HyperlinkedModelSerializer
# --------------------------------------------

def raise_errors_on_nested_writes(method_name, serializer, validated_data):
    """
    Give explicit errors when users attempt to pass writable nested data.

    If we don't do this explicitly they'd get a less helpful error when
    calling `.save()` on the serializer.

    We don't *automatically* support these sorts of nested writes because
    there are too many ambiguities to define a default behavior.

    Eg. Suppose we have a `UserSerializer` with a nested profile. How should
    we handle the case of an update, where the `profile` relationship does
    not exist? Any of the following might be valid:

    * Raise an application error.
    * Silently ignore the nested part of the update.
    * Automatically create a profile instance.
    """

    # Ensure we don't have a writable nested field. For example:
    #
    # class UserSerializer(ModelSerializer):
    #     ...
    #     profile = ProfileSerializer()
    assert not any(
        isinstance(field, BaseSerializer) and
        (field.source in validated_data) and
        isinstance(validated_data[field.source], (list, dict))
        for field in serializer._writable_fields
    ), (
        'The `.{method_name}()` method does not support writable nested '
        'fields by default.\nWrite an explicit `.{method_name}()` method for '
        'serializer `{module}.{class_name}`, or set `read_only=True` on '
        'nested serializer fields.'.format(
            method_name=method_name,
            module=serializer.__class__.__module__,
            class_name=serializer.__class__.__name__
        )
    )

    # Ensure we don't have a writable dotted-source field. For example:
    #
    # class UserSerializer(ModelSerializer):
    #     ...
    #     address = serializer.CharField('profile.address')
    assert not any(
        '.' in field.source and
        (key in validated_data) and
        isinstance(validated_data[key], (list, dict))
        for key, field in serializer.fields.items()
    ), (
        'The `.{method_name}()` method does not support writable dotted-source '
        'fields by default.\nWrite an explicit `.{method_name}()` method for '
        'serializer `{module}.{class_name}`, or set `read_only=True` on '
        'dotted-source serializer fields.'.format(
            method_name=method_name,
            module=serializer.__class__.__module__,
            class_name=serializer.__class__.__name__
        )
    )
```
简单来说就是官方觉得在嵌套写的情况下，在更新某实例的时候，如果出现一个不存在的被嵌套实例，那么以下三种方式都认为是合理的

- 报错
- 忽略掉这个被嵌套的实例
- 创建一个被嵌套的实例
- ...

这还只是举的一个小例子，实际情况更加复杂，因此官方直接禁止了嵌套写

## 解决方案
有两种解决方案，一种是写的时候不要用嵌套，在前端做好判断工作，如果你要更新用户名字，那就将这个嵌套序列化变成只读的，如果你想更新邮箱，直接更新邮箱
``` python
class UserSerializers(ModelSerializer):
    # 这里将嵌套Serilizer设为只读的
    email = EmailSerializers(many=True, read_only=True)
    class Meta:
        model = User
```
应该说这种解决方案是最优雅的，因为完全贴合了官方的意思，但是在实际开发过程中，万一前端不干呢？等于你将一堆任务扔给了前端做，如果提前在接口文档里面说好还好（前提是你预先就知道不能嵌套写，不过你要是预先知道也不会来看我这篇文章了哈哈哈），你中途发现这个问题再跟前端说前端可能会找你拼命。还有在某些具体案例中可能很难判断，必须要进行嵌套写，那么就是我下面说的方案了：**`explicit method`**

首先明确，`create`函数对应创建，`update`函数对应更新，废话不多说，直接上代码
``` python
class UserSerializers(ModelSerializer):
    # 这里增加一个Serilizer
    email = EmailSerializers(many=True)
    class Meta:
        model = User

    def create(self, validated_data):
        # 先将email字段pop出来备用
        emails = validated_data.pop("email")
        instance = User.objects.create(**validated_data)
        
        # 然后再存储email
        if emails is None:
            return instance
        for email in emails:
            email["user"] = instance
            Email.objects.create(**email)
        return instance

    def update(self, instance, validated_data):
        instance.name = validated_data["name"]

        instance.save()

        for item in validated_data["email"]:
            item_instance = Email.objects.get(id=item["id"])
            item_serializer = EmailSerializers(item_instance, data=item, partial=True)
            item_serializer.is_valid(raise_exception=True)
            item_serializer.save()
        return instance
```
代码相当通俗易懂，这里我们展示了一个简单的写法，实际案例中还需要具体分析