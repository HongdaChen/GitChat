### 前言

**Keller 云笔记** 支持的笔记格式有两种：富文本笔记和 MarkDown 笔记，上篇我们实现了富文本笔记的编辑和展示，本篇，我们将实现
MarkDown 笔记的编辑和展示。

### 知识准备

编辑 MarkDown 笔记，推荐大家使用 **mavon-editor** 编辑器：简洁、易用、完全支持 Vue.js。

#### **mavon-editor**

认识 **mavon-editor** 从它的使用效果开始：

![MarkDown
使用效果](https://images.gitbook.cn/8ea29b10-8958-11ea-8d82-3f842d05da77)

从图中可以看到，它支持 MarkDown 的所有语法格式，同时支持分屏和全屏模式，是一款不可多得的写作神器。文档地址：

> <https://www.npmjs.com/package/mavon-editor>

![MarkDown](https://images.gitbook.cn/9c9b9230-8958-11ea-91b0-1940e1c2e5e5)

Keller 云笔记中使用 npm 命令安装 mavon-editor 即可：

    
    
    npm install mavon-editor
    

mavon-editor 支持快捷键比较丰富：

key | keycode | 功能  
---|---|---  
F8 | 119 | 开启/关闭导航  
F9 | 120 | 预览/编辑切换  
F10 | 121 | 开启/关闭全屏  
F11 | 122 | 开启/关闭阅读模式  
F12 | 123 | 单栏/双栏切换  
TAB | 9 | 缩进  
CTRL + S | 17 + 83 | 触发保存  
CTRL + D | 17 + 68 | 删除选中行  
CTRL + Z | 17 + 90 | 上一步  
CTRL + Y | 17 + 89 | 下一步  
CTRL + BreakSpace | 17 + 8 | 清空编辑  
CTRL + B | 17 + 66 | **加粗**  
CTRL + I | 17 + 73 | _斜体_  
CTRL + H | 17 + 72 | # 标题  
CTRL + 1 | 17 + 97 or 49 | # 标题  
CTRL + 2 | 17 + 98 or 50 | ## 标题  
CTRL + 3 | 17 + 99 or 51 | ### 标题  
CTRL + 4 | 17 + 100 or 52 | #### 标题  
CTRL + 5 | 17 + 101 or 53 | ##### 标题  
CTRL + 6 | 17 + 102 or 54 | ###### 标题  
CTRL + U | 17 + 85 | ++下划线++  
CTRL + M | 17 + 77 | ==标记==  
CTRL + Q | 17 + 81 | > 引用  
CTRL + O | 17 + 79 | 1\. 有序列表  
CTRL + L | 17 + 76 | 链接  
CTRL + ALT + S | 17 + 18 + 83 | ^上角标^  
CTRL + ALT + U | 17 + 18 + 85 | \- 无序列表  
CTRL + ALT + C | 17 + 18 + 67 | ``` 代码块  
CTRL + ALT + L | 17 + 18 + 76 | 图片链接  
CTRL + ALT + T | 17 + 18 + 84 | 表格  
CTRL + SHIFT + S | 17 + 16 + 83 | ~下角标~  
CTRL + SHIFT + D | 17 + 16 + 68 | 中划线  
CTRL + SHIFT + C | 17 + 16 + 67 | 居中  
CTRL + SHIFT + L | 17 + 16 + 76 | 居左  
CTRL + SHIFT + R | 17 + 16 + 82 | 居右  
SHIFT + TAB | 16 + 9 | 取消缩进  
  
### 业务流程

MarkDown 笔记业务笔记预览效果如下：

![MarkDown流程](https://images.gitbook.cn/de135e00-8958-11ea-8e4b-3d12ac21b89d)

对应的业务流程说明：

  1. 用户选择打开一个笔记
  2. 当选择的是一个 MarkDown 笔记时，在右侧展示该笔记的内容，同时右上角有一个编辑图标
  3. 用户点击编辑图标，加载 mavon-editor 编辑器
  4. 编辑过程中，用户可以点击右上角的保存图标，保存笔记

### 前端代码实现

MarkDown 笔记的读取和保存在服务端上和富文本笔记是一样的，使用相同的接口即可，只需要实现前端代码。涉及到的代码结构如下：

  * components
    * Note.vue：自定义组件，处理笔记详情
    * MarkDown.vue：自定义组件，封装 mavon-editor 组件，用于编辑 MarkDown 笔记内容

#### **自定义 MarkDown 组件**

mavon-editor 的使用流程是：

  * 引入组件
  * 引入 css 样式文件
  * 在页面中使用 mavon-editor 组件，需要处理的属性和方法如下
    * v-model：绑定的数据模型，这里要传入的是 MarkDown 格式的文本内容
    * ref：定义该属性的值，用于获取到 mavon-editor dom 节点
    * @imgAdd：添加图片时的处理方法
    * @change：编辑器内容有变化时的回调方法

新建 MarkDown.vue 文件，代码如下：

    
    
    <!-- 将 mavon-editor 组件包装下，作为 markdown 编辑器插件 -->
    <template>
        <div>
            <mavon-editor v-model="value" ref="md" @imgAdd="imgAdd" @change="change"  style="min-height: 600px" />
        </div>
    </template>
    <script>
        import {mavonEditor} from 'mavon-editor';
        import 'mavon-editor/dist/css/index.css';
        export default {
            components: {
                mavonEditor
            }
        }
    </script>
    

@imgAdd 回调方法绑定的处理函数如下：

    
    
    // 将图片上传到服务器，返回地址替换到 md 中
                imgAdd(pos, $file) {
    
                    /**
                     * 服务端接口用 MultipartFile file 参数接收的，此处用 formdata.append('file', $file) 传
                     */
                    let formdata = new FormData();
            // 设置上传参数中图片的参数名为 file
                    formdata.append('file', $file);
            // 设置 token 参数
                    formdata.append('token', window.localStorage.getItem("token"));
                    // 使用 axios 发送 post 请求
                    axios.post(editorImgUploadUrl, 
                        formdata
                    , {
                        headers: {
                            'Content-Type': 'multipart/form-data'
                        }
                    }).then(response => {
                        let {
                            success,
                            message,
                            data
                        } = response.data;
    
                        //应答不成功，提示错误信息
                        if (success !== 0) {
                            this.$message({
                                message: message,
                                type: 'error'
                            });
                        } else {
                // 上传完成后，将服务端返回的图片地址设置到编辑器中
                            this.$refs.md.$img2Url(pos, data);
                        }
                    }).catch(err => {
                        this.$notify.error({
                            title: 'Failed',
                            message: err
                        });
                    });
    
                }
    

另外，mavon-editor 还支持删除图片的回调 @imgDel，在这里做简单介绍，大家可以自己实现一下：

  * 在编辑器中删除一个图片时的回调方法
  * 回调的参数为：`String: filename`
  * 可以为这个回调方法添加相应的处理函数，根据图片名称，从文件服务器中删除对应的图片。这样可以避免文件服务器中存储大量没有用到的图片

至此，自定义的 MarkDown 组件基本定义完成，为其添加 getText()、getHtml()、load() 方法，供父组件调用：

    
    
                /**
                 * 获取文本内容
                 */
                getText() {
                    return this.value;
                },
                /**
                 * 获取解析后的 Html 内容
                 */
                getHtml() {
                    return this.render;
                },
                //设置 rander 的值是为了保证当编辑器内容没有变化时能取到原来的值
                load(value, render) {
                    this.value = value;
                    this.render = render;
                }
    

#### **在 Note 中使用 MarkDown 组件**

在 Note.vue 中

  * 第一步：引入 MarkDown

    
    
    import WangEditor from "./WangEditor.vue"
        import MarkDown from "./MarkDown.vue"
    
        import {
            req_getNoteInfoById,
            req_setNoteContent
        } from "../api.js";
        export default {
            components: {
                WangEditor,MarkDown
            },
    

  * 第二步，在页面中使用 MarkDown

    
    
    <MarkDown v-show = "editMode && note.type == 1" ref = "markDown" ></MarkDown>
    

  * 第三步，添加相应处理方法

切换到编辑模式时，判断是 MarkDown 笔记，则加载 MarkDown 组件：

    
    
    //富文本笔记加载 WangEditor
                    if (this.note.type == 0) {
                        this.$refs.wangEditor.load(this.note.html);
                    }
                    // MarkDown 笔记加载 mavon-editor
                    else if(this.note.type == 1){
                        this.$refs.markDown.load(this.note.text,this.note.html);
                    }
    

保存时，判断是 MarkDown 笔记，调用 MarkDown 组件的方法获取相应内容：

    
    
    handleSave() {
                    let newText;
                    let newHtml;
                    if(this.note.type == 0){
                        newText = this.$refs.wangEditor.getText();
                        newHtml = this.$refs.wangEditor.getHtml();
                    }else if(this.note.type == 1){
                        newText = this.$refs.markDown.getText();
                        newHtml = this.$refs.markDown.getHtml();
                    }else{
                        return;
                    }
          ... ...
    }
    

#### **Ctrl+S**

mavon-editoer 编辑器支持丰富的快捷键，其中 `Ctrl+S` 需要我们处理一下回调方法 @Save：

  * @save：保存内容的回调
    * 点击菜单栏保存图标和快捷键 `Ctrl+S` 触发的回调方法
    * 回调的参数为：`String: value`，`String: render`
    * 可以为这个回调方法添加相应的处理函数，这样就可以实现笔记随时保存的功能了

使用 `Ctrl+S` 保存笔记内容需要三步：

  * 第一步： `<mavon-editor>` 标签中添加 @save 回调方法的处理函数 save

    
    
            <mavon-editor v-model="value" ref="md" @save="save"  />
    

  * 第二步：在 save 函数中保存当前文档的内容，并回调到父组件的方法

    
    
                save(value, render) {
                    // render 为 markdown 解析后的结果[htmlStr]
                    this.value = value;
                    this.render = render;
                    this.func();
                }，
                func() {
                    this.$emit('func');
                }
    

  * 第三步：在父组件中处理回调方法，将 handleSave 函数绑定到 MarkDown 组件的回调方法中即可

    
    
    <MarkDown v-show = "editMode && note.type == 1" ref = "markDown" @func="handleSave"></MarkDown>
    

### 效果测试

编辑过程中随时用 `Ctrl+S` 保存笔记内容：

![Ctrl](https://images.gitbook.cn/81fde1f0-895b-11ea-9db6-d98fa851b4fc)

### 源码地址

本篇完整的源码地址如下：

> <https://github.com/tianlanlandelan/KellerNotes>

### 小结

本篇使用 mavon-editor 编辑器完成 MarkDown 笔记的编辑功能，同时添加了 `Ctrl+S`
快捷键支持。至此，云笔记用户的基本功能设计完毕，接下来进行项目的打包和发布工作。

