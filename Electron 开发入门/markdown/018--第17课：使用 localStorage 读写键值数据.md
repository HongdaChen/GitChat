对于大多数桌面应用来说，都绕不开数据存储，我们可以将数据保存在各种类型的文件或数据库中，如纯文本文件、二进制文件、XML 文件、JSON
文件、关系型数据库、文档数据库等。

由于 Electron 应用本质上是基于 Web 技术的，因而 Electron 应用的存储方案其实就是 Web 的存储方案。在以前，Web
存储方案非常单一，数据都是依赖于后端数据库的存储，如果用户在前端页面输入一些数据，提交后，数据会被保存到服务端的数据库中（如 MySQL、SQL
Server 等）。不过随着 HTML 5 的兴起，Web 可以将数据保存到前端，这样 Web 数据存储方案就分为前端和后端。

由于 Electron 同时也可以调用 Node.js API，因而也支持 SQLite 数据库，不过 Electron 中操作 SQLite
数据库有一些特别，这一点在后面的文章中会详细介绍。

本节会介绍一种比较简单的键值存储技术 localStorage，它属于浏览器 API，并不需要使用第三方组件。

localStorage 会使用 key/value 的方式存储数据，本例会使用 localStorage
技术实现一个笔记本的应用。在笔记本中输入的文本会实时存储在 localStorage 中，下面是实现步骤。

（1）实现主页面（index.html）

    
    
    <html>
        <head>
            <title>笔记本</title>
            <link rel="stylesheet" type="text/css" href="index.css">
            <script src="event.js"></script>
        </head>
        <body>
            <div id="close" onclick="quit();">x</div>
            <textarea id="textarea" onKeyUp="saveNotes();"></textarea>
        </body>
    </html>
    

在 index.html 页面中放置了一个 `<div>` 标签和一个 `<textarea>` 标签。`<div>`
标签是一个关闭按钮，显示在页面的右上角。`<textarea>` 标签用于输入笔记内容，并指定了 onKeyUp
事件，一旦有键盘动作（输入了任何字符），就会立刻调用 saveNotes() 函数将整个笔记内容重新保存到 localStorage
中。笔记页面的效果如下。

Mac OS X 的效果。

![](https://images.gitbook.cn/84248800-85cc-11e9-aa05-69a3b0b89a04)

Windows 的效果。

![](https://images.gitbook.cn/8cedfb60-85cc-11e9-9551-d533637b7472)

（2）主页面样式（index.css）

    
    
    body {
        background: #E1FFFF;
        color: #694921;
        padding: 1em;
    }
    
    textarea {
        font-family: 'Hannotate SC', 'Hanzipen SC','Comic Sans', 'Comic Sans MS';
        outline: none;
        font-size: 18pt;
        border: none;
        width: 100%;
        height: 100%;
        background: none;
    }
    
    #close {
        cursor: pointer;
        position: absolute;
        top: 8px;
        right: 10px;
        text-align: center;
        font-family: 'Helvetica Neue', 'Arial';
        font-weight: 400;
    }
    

在 index.css 样式文件中设置了 body 背景和文字颜色，文本输入区的字体，关闭按钮的位置等属性。

（3）编写 event.js 脚本文件

    
    
    const electron = require('electron');
    const app = electron.remote.app;
    //初始化页面
    function initialize () {
            //从 localStorage 中获取保存的笔记
        let notes = window.localStorage.notes;
        if (!notes) notes = '记录生活的点点滴滴...';
           //将保存的笔记显示在文本输入区域
           textarea.value = notes;
    }
    function saveNotes () {
        let notes = textarea.value;
            //保存输入的笔记
        window.localStorage.setItem('notes',notes);
    }
    //退出笔记本
    function quit () { app.quit(); }
    window.onload = initialize;
    

[点击这里下载源代码](https://github.com/geekori/electron_gitchat_src)

### 答疑与交流

为了让订阅课程的读者更快更好地掌握课程的重要知识点，我们为每个课程配备了课程学习答疑群服务，邀请作者定期答疑，尽可能保障大家学习效果。同时帮助大家克服学习拖延问题！

请添加小助手伽利略微信 GitChatty6，并将支付截图发给她，小助手会拉你进课程学习群。

