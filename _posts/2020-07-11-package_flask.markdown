---
layout: post
title: '打包flask应用'
subtitle:   "package flask server"
date:       2020-07-11 00:11:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - flask
---

## 打包
实验代码是最简单的flask工程，"/", render_template到hello.html.
hello.html中渲染"hello world"
使用`pyinstaller -F app.py --name app` 对服务进行打包，在dist中发现app.exe文件
但是执行时，出现如下错误

![package_call_error](\img\package_flask\\package_error.png)

搜索问题的答案[stackoverflow] (https://stackoverflow.com/questions/32149892/flask-application-built-using-pyinstaller-not-rendering-index-html/48976223#48976223)

## 修改
添加如下代码在flask入口最上面的import下，sys._MEIPASS是打包目录
```python
if getattr(sys, 'frozen', False):
  template_folder = os.path.join(sys._MEIPASS, 'templates')
  app = Flask(__name__, template_folder=template_folder)
else:
  app = Flask(__name__)
```
执行` pyinstaller  -F --add-data "templates;templates"  app.py` 进行打包。
打包完成后，执行dist下app.exe，发送请求,执行成功

![hello_world](\img\package_flask\\hello_world.png)
