---
layout: post
title: 'Flask 反序列化对象未绑定session'
subtitle:   "Flask deserialize object not bond to session"
date:       2020-07-12 11:30:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - flask
---
## Flask反序列化对象
从redis缓存中取出Artcile对象，获取文章对应作者，如下
```python
res.append(
    {
        "id": article.id ,
        "title": "<a href='%s' style='cursor:pointer'>%s</a>"
                 % (article.id,article.title),
        "auther": article.user.name, # 此处异常
        
        'edit': "<a href='edit/%s' style='cursor:pointer' >编辑<a>" % article.id
     }
```
Flask反序列化对象时出错**sqlalchemy.orm.exc.DetachedInstanceError**"Parent instance <Article at 0x1a72390e5c8> is not bound to a Session; lazy load operation of attribute 'user' cannot proceed"
![not_bond_error](/img/flask_serialize_not_bond_to_session/flask_serialize_not_bond_to_session_error.png)
##原因
在数据模型中Article和User.id外键关联,sqlachemy默认使用懒加载，不会加载Article关联的user信息。
当使用session查询数据库时，不会有问题。从缓存中序列化出来的Article对象时，访问user对象时候，使用默认访问数据库的session对象，user未绑定到session，导致查询时失败。
```python
class Article(db.Model):
    __tablename__ = 'article'
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String, nullable=False)
    body = db.Column(db.String, nullable=False)
    body_html = db.Column(db.String)
    created = db.Column(db.DateTime, nullable=False, default=datetime.now)
    # 每篇文章都有一个作者，一一对应关系,在jinja模板中可以使用article.user.name获取用户名
    
    auther_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    # 每篇文章可以有n条评论，一对多关系
    
    comments = db.relationship('Comment', backref='article')
```
## 解决方法
进行缓存时，关联查询user对象(不使用懒加载)，进行存储。
```python
# articles = Article.query.all()

articles = Article.query.options(db.joinedload("user")).all()
```

