---
layout:       post
title:        "django-practice（二）"
date:         2017-04-05 12:00:00
categories: document
tag:
  - django
  - python
---

* content
{:toc}


### ORM
 一对一、多对多
+ 一对一： 一般用于某张表的补充，比如用户基本信息是一张表，但并非每一个用户都需要有登陆的权限，不需要记录用户的用户名和密码，此时，合理的做法就是新建一张记录登录信息的表，与用户信息进行一对一的关联，可以方便的从不查询母表信息或反向查询。
+ 外键： 有很多的应用场景，比如每一个员工归属于一个部门，那么就可以让员工表的部门字段与部门表进行一对多的关联，可以查询到一个员工属于哪个部门，也可反向查询某一部门有哪些员工
+ 共同点： 在admin中添加数据的话，都会出现在一个select选框，但只能单选，因为不论一对一还是一对多，自己都是"一"
+ 多对多： 比如有多个孩子，和多种颜色，每个孩子可以喜欢多种颜色，每个颜色可以有多个孩子喜欢，对于双向均是可以有多个选择

### form表单验证和字段验证
在调用is_valid()方法时执行
+ 字段的to_python()方法是验证第一步。它将值强制转化为正确的数据类型，如果不能转换则引发validatiionError。 这个方法从Widget 接收原始的值并返回转换后的值。例如，FloatField 将数据转换为Python 的float 或引发ValidationError。
+ 字段validate()方法处理字段特异性的验证，这种验证不适合位于validator口。它接收一个已经转换成正确的数据类型的值，并在发现错误时引发ValidationError。这个方法不返回任何东西且不应该改变任何值。
+ run_validators
+ field子类的clean（）方法。 这个方法可以返回一个完全不同的字典，该字典将用作cleaned_data。它负责以正确的顺序to_python、 validate、 run_validators并传播他们的错误。这个方法返回验证后的数据，这个数据在后面将插入到表单的clean_data字典中。
+ 表单子类中的clean_<fieldname>() 方法 —— <fieldname> 通过表单中的字段名称替换。这个方法完成于特定属性相关的验证，这个验证与字段的类型无关。这个方法没有任何传入的参数。你需要查找self.cleaned_data 中该字段的值，记住此时它已经是一个Python 对象而不是表单中提交的原始字符串（它位于cleaned_data 中是因为字段的clean() 方法已经验证过一次数据）。例如，如果你想验证名为serialnumber 的CharField 的内容是否唯一， clean_serialnumber() 将是实现这个功能的理想之处。你需要的不是一个特别的字段（它只是一个CharField），而是一个特定于表单字段特定验证，并规整化数据。这个方法返回从cleaned_data 中获取的值，无论它是否修改过。
+ 表单字类的clean()方法。这个方法可以实现需要同时访问表单多个字段的验证。这里你可以验证如果提供字段A，那么字段B 必须包含一个合法的邮件地址以及类似的功能。还要注意，覆盖ModelForm 子类的clean() 方法需要特殊的考虑。（更多信息参见ModelForm) [http://python.usyiyi.cn/translate/django_182/topics/forms/modelforms.html#overriding-modelform-clean-method]文档）。

这些方法按以上给出的顺序执行，一次验证一个字段。也就是说，对于表单中的每个字段（按它们在表单定义中出现的顺序），先运行Field.clean() ，然后运行clean_<fieldname>()。每个字段的这两个方法都执行完之后，最后运行Form.clean() 方法，无论前面的方法是否抛出过异常。

### 字段类型

+ Ip GenericIPAddressField
+ FileField
+ GenericIPAddressField
+ CharField


### Bootstrap CSS
CSS:cascading style sheets:层叠样式表，用来描述使用标记语言写成的网站的外观和格式。

### models
### 表单
Django表单要么是绑定的(bound)，要么是未绑定的，如果是绑定的，那么它能够验证数据，并渲染表单及其数据成html；若是未绑定的，那么它就不能够完成验证（因为没有可验证的数据），但是仍能渲染空白 的表单成html
+ 创建未绑定的表单实例，只需简单的实例化该类 `f = ContactForm()`
+ 若要绑定数据到表单，可以将数据以字黄的形式传递给表单类的构造函数的第一个参数。
```
data={'subject':'hello','message':'hi,there','cc_myslf':'True'}
f = ContactForm(data)
```
在这个字典中，键为字段的名称，它们对应于表单类中的属性。值为需要验证的数据。它们通常是字符串，但是没有强制 要求必须是字符串。传递的数据类型取决于字段。
+ 如果你要确认是否已绑定表单，可以检查下表单is_bound属性的值
```
f=ContactForm()
f.is_bound
>> False
f=ContactForm('subject':'hello')
>>True
```
注意，传递一个空的字典将创建一个带有空数据的绑定的表单：
```
f=ContactForm({})
f.is_bound
>>True
```
#### 使用表单来验证数据
+ Form.clean()
当你需要为相互依赖的字段添加自定义的验证时，你可以实现表单的clean()方法
+ Form.is_valid()
表单对象的首要任务是验证数据。对于绑定的表单实例，可以调用is_valid()方法来执行验证并返回一个表示数据是否合法的布尔值。
+ 处理常规的表单数据，应该使用HttpRequest.POST
### request
+ request.method
```
if request.method == 'GET':
    do_something()
elif request.method == 'POST':
    do_something_else()
```
+ request.POST.get
+ request.FILES
key-value形式，key为name 如```<input type="file" name="" />``` value为FIFLES里上传的uploadfile
>>> 注意，files 只有在method为POST且<form>里包含 ```enctype="multipart/form-data"```
