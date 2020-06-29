---
layout: post
title: 'flask sqlachemy时间格式'
subtitle:   "flask sqlachemy time format"
date:       2020-06-29 21:30:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - flask
---
## 问题
在flask-sqlachemy中，使用sqlachemy创建created字段
```python
created = db.Column(db.DateTime,  nullable=False, default=datetime.now)
```
但是页面上读取数据库存入时间是


![错误时间](/img/in-post/Flask-Sqlachemy-Time/1593436434901.png)


## 修改
查询数据类型，Datetime是肯定带毫秒，除非使用String，存储时将当前时间格式化后存储。

![数据模型](/img/in-post/Flask-Sqlachemy-Time/1593438899112.png)
简单做法是定义jinja2中过滤方法，如下
```python
@app.template_filter('strftime')
def datetime_format(date):
    return date.strftime("%Y-%m-%d %H:%M:%S")
```
在html中使用`strftime`
```
<span>作者:\{\{ article.user.name \}\}</span> <span>发表日期:\{\{ article.created | strftime \}\}</span>
```
效果如下：

![正确时间](/img/in-post/Flask-Sqlachemy-Time/1593442644149.png)


