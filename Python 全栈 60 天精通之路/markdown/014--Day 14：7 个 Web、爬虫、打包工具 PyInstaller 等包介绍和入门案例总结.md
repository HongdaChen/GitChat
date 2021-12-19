Python 拥有许多强大的扩展包，为 Web 开发者、数据分析从业人员、机器学习工程师，快速构建模型提供便利。

下面介绍 7 个 Web、爬虫、打包相关的工具包。

### Django

Django 是最通用的 Web 开发框架之一，可以帮助开发者从零创造一个全功能的大型 Web 应用程序。

详情参考：

> <https://www.djangoproject.com/>

![](https://images.gitbook.cn/2020-02-06-013424.png)

### Flask

Flask 是一个轻量级的 WSGI Web 应用框架，适合搭建轻量级的 Web 应用程序，详情参考：

> <https://palletsprojects.com/p/flask/>

Flask 是 Python 轻量级 Web 框架，容易上手，被广大 Python 开发者所喜爱。

![](https://images.gitbook.cn/2020-02-06-013425.png)

下面是 Flask 版 hello world 的构建过程。

首先 `pip install Flask` 安装 Flask，然后 import Flask，同时创建一个 app。

    
    
    from flask import Flask
    
    App = Flask(__name__)
    

写一个 index 页的入口函数，返回 `hello world`。通过装饰器 `App.route('/')` 创建 index 页的路由，一个 `/`
表示 index 页：

    
    
    @App.route('/')
    def index():
        return "hello world"
    

调用 index 函数：

    
    
    if __name__ == "__main__":
        App.run(debug=True)
    

然后启动，会在 console 下看到如下启动信息，表示服务启动成功。

    
    
    * Debug mode: on
    * Restarting with stat
    * Debugger is active!
    * Debugger PIN: 663-788-611
    * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
    

接下来，打开一个网页，相当于启动客户端。

在 URL 栏中输入：

    
    
    http://127.0.0.1:5000/
    

看到页面显示 `hello world`，证明服务访问成功。

同时，在服务端后台，看到如下信息，表示处理一次来自客户端的 GET 请求。

    
    
     27.0.0.1 - - [03/Feb/2020 21:26:50] "GET / HTTP/1.1" 200 -
    

### FastAPI

FastAPI 是一个现代、高性能 Web 框架，用于构建 APIs，基于 Python 3.6 及以上版本。

最大特点：快！性能极高，可与 Node.js、Go 媲美。

基于 Starlette 和 Pydantic，是 FastAPI 如此高性能的重要原因。还具备代码复用性高、容易上手、健壮性强的优点。

FastAPI 还有一个非常强的优势：方便的 API 调试，生成 API 文档，直接能够做到调试自己构建的 API，这在实际应用中，价值凸显。

![](https://images.gitbook.cn/2a984e10-5584-11ea-afc1-9b7d03147fc6)

入门案例：

    
    
    from fastapi import FastAPI
    from pydantic import BaseModel
    
    app = FastAPI()
    
    class User(BaseModel):
        id: int
        name: str
        friends: list
    
    
    @app.get("/")
    def index():
        return {"admin": "welcome to FastAPI"}
    
    
    @app.get("/users/{user_id}")
    def read_user(user_id: int, name: str = None):
        return {"user_id": user_id, "name": name}
    
    
    @app.put("/users/{user_id}")
    def update_user(user_id: int, user: User):
        return {"user_name": user.name, "user_id": user_id}
    

将上述代码保存为 main.py，再安装与构建服务相关的框架 uvicorn。

安装完成后，后台执行：

    
    
    uvicorn main:app --reload
    

启动服务，显示如下：

![](https://images.gitbook.cn/945d8670-5585-11ea-a2ae-1b10e8f1b0a0)

打开客户端，输入 `localhost:8000`，回车：

![](https://images.gitbook.cn/cb387e70-5585-11ea-8ccf-d385e6cc64ec)

输入请求 `localhost:8000/users/5`，回车，看到前台数据，非常方便的就能传递到 controller 层。

![](https://images.gitbook.cn/f71792b0-5585-11ea-afc1-9b7d03147fc6)

输入请求 `localhost:8000/docs`，回车；看到 API 文档界面：

![](https://images.gitbook.cn/20ac41c0-5586-11ea-908b-b5332759803b)

点开第二个 GET 请求，然后点击“Try it out”后，便可以进行接口调试。非常方便！

![](https://images.gitbook.cn/520ddbc0-5586-11ea-9dfa-cf3c87b75c44)

输入 user_id、name 后，点击 Execute，执行成功。

如果 user_id 输入非数值型，点击 Execute 后，红框闪动一下，不会执行，直到输入正确为止。

![](https://images.gitbook.cn/7327aca0-5586-11ea-bb07-d79dc32970fa)

输入 user_id、name 后，点击 Execute，能看到结果，包括请求的 URL：

![](https://images.gitbook.cn/b57b3270-5586-11ea-8ccf-d385e6cc64ec)

也能看到，服务器响应前端，返回的结果：

![](https://images.gitbook.cn/0c963d20-5587-11ea-987f-4debc547abef)

FastAPI 基于以上这些强大的优点，在实际开发 API 服务时，会很敏捷。

### Requests

Requests 生成、接受、解析一个 HTTP 请求，使用 Requests 做这些事情都非常简单。

详情参考：

> <https://2.python-requests.org/en/master/>

![](https://images.gitbook.cn/2020-02-06-013431.png)

举个案例，爬取北京今日天气。

![](https://images.gitbook.cn/2020-02-06-013434.png)

    
    
    import requests
    
    url = 'http://www.weather.com.cn/weather1d/101010100.shtml#input'
    with requests.get(url) as res:
        status = res.status_code
        print(status)
        content = res.content
        print(content)
    

返回的结果：

    
    
    200
    ISO-8859-1
    # 网页的内容
    

然后结合下面介绍的 lxml 库，提取想要的标签值

### lxml

lxml 是 Python 很好用的处理 XML 和 HTML 数据的库。

详情参考：

> <https://lxml.de/>

![1574589725462](https://images.gitbook.cn/2020-02-06-013445.png)

爬取的 HTML 网页结构，如下所示：

![](https://images.gitbook.cn/2020-02-06-013446.png)

    
    
    import requests
    from lxml import etree
    import pandas as pd
    import re
    
    url = 'http://www.weather.com.cn/weather1d/101010100.shtml#input'
    with requests.get(url) as res:
        content = res.content
        html = etree.HTML(content)
    

通过 lxml 模块提取值：

    
    
    location = html.xpath('//*[@id="around"]//a[@target="_blank"]/span/text()')
    temperature = html.xpath('//*[@id="around"]/div/ul/li/a/i/text()')
    

结果：

    
    
    ['香河', '涿州', '唐山', '沧州', '天津', '廊坊', '太原', '石家庄', '涿鹿', '张家口', '保定', '三河', '北京孔庙', '北京国子监', '中国地质博物馆', '月坛公
    园', '明城墙遗址公园', '北京市规划展览馆', '什刹海', '南锣鼓巷', '天坛公园', '北海公园', '景山公园', '北京海洋馆']
    
    ['11/-5°C', '14/-5°C', '12/-6°C', '12/-5°C', '11/-1°C', '11/-5°C', '8/-7°C', '13/-2°C', '8/-6°C', '5/-9°C', '14/-6°C', '11/-4°C', '13/-3°C'
    , '13/-3°C', '12/-3°C', '12/-3°C', '13/-3°C', '12/-2°C', '12/-3°C', '13/-3°C', '12/-2°C', '12/-2°C', '12/-2°C', '12/-3°C']
    

### Pillow

Pillow 是一个编辑图像的处理库。可用来创建复合图像、应用过滤器、修改透明度、转换图像文件类型等。

详情参考：

> <https://pillow.readthedocs.io/en/stable/>

![](https://images.gitbook.cn/2020-02-06-013435.png)

首先，安装 Pillow：

    
    
    pip install pillow
    

然后，导入待处理的图像：

    
    
    from PIL import Image
    im = Image.open('./img/plotly2.png')
    

![](https://images.gitbook.cn/2020-02-06-013437.png)

旋转图像 45 度：

    
    
    im.rotate(45).show()
    

![](https://images.gitbook.cn/2020-02-06-013439.png)

过滤图像：

    
    
    from PIL import ImageFilter
    im.filter(ImageFilter.CONTOUR).show()
    

![](https://images.gitbook.cn/2020-02-06-013442.png)

等比例缩放图像：

    
    
    im.thumbnail((128,72),Image.ANTIALIAS).show()
    

![image-20200205161724868](https://images.gitbook.cn/2020-02-06-013440.png)

### PyInstaller

PyInstaller 能将一个应用程序打包为独立可执行的文件。比如 Windows 下打包为 EXE 文件。

详情可参考：

> <http://www.pyinstaller.org/>

![](https://images.gitbook.cn/2020-02-06-013448.png)

PyInstaller 相比于同类的优势：

  * 支持 Python 2.7、Python 3.3-3.6
  * 生成的可执行文件字节数更小
  * 对第三方包的支持非常好，只需要将它们放到 Python 的解释器对应的文件夹中，PyInstaller 便可自动打包到最终生成的可执行文件中。

PyInstaller 的详细打包过程会单独有一篇文章总结。

### Pydantic

FastAPI 基于 Pydantic，Pydantic 主要用来做类型强制检查。参数赋值，不符合类型要求，就会抛出异常。

对于 API 服务，支持类型检查非常有用，会让服务更加健壮，也会加快开发速度。开发者再也不用自己写一行一行的代码，去做类型检查。

首先，

    
    
    pip install pydantic
    

然后，使用 Pydantic 做强制类型检查。

    
    
    from pydantic import ValidationError
    
    from datetime import datetime
    from typing import List
    from pydantic import BaseModel
    
    class User(BaseModel):
        id:int
        name='jack guo'
        signup_timestamp: datetime = None
        friends: List[int] = []
    

观察到：

  * id 要求必须为 int
  * name 要求必须为 str，且有默认值
  * signup_timestamp 要求为 datetime，默认值为 None
  * friends 要求为 List，元素类型要求 int，默认值为 []

使用 User 类：

    
    
    try:
        User(signup_timestamp='not datetime',friends=[1,2,3,'not number'])
    except ValidationError as e:
        print(e.json())
    

  * id 没有默认值，按照预期会报缺失的异常
  * signup_timestamp 被赋为非 datetime 类型值，按照预期会报异常
  * friends 索引为 3 的元素被赋值为 str，按照预期也会报异常

执行代码，验证是否符合预期。执行结果显示，符合预期：

    
    
    [
      {
        "loc": [
          "id"
        ],
        "msg": "field required",     
        "type": "value_error.missing"
      },
      {
        "loc": [
          "signup_timestamp"
        ],
        "msg": "invalid datetime format",
        "type": "value_error.datetime"
      },
      {
        "loc": [
          "friends",
          3
        ],
        "msg": "value is not a valid integer",
        "type": "type_error.integer"
      }
    ]
    

### 小结

今天，与大家一起学习 8 个 Web、爬虫和打包相关的包和框架。基于这些包开发，就像是站在巨人肩上，开发效率会倍增。

