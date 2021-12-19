### 前言

通过前面两篇的学习，我们实现了在用户主页显示用户笔记本列表，并且在每次选中新的笔记本时展示笔记本中的笔记。接下来，我们将展示并编辑用户的笔记，本篇将实现富文本类型的笔记的编辑和展示功能。

### 知识准备

编辑富文本笔记，就需要了解一下支持 Vue.js 的富文本笔记组件。百度一下、Google
一下都能发现很多优秀的编辑器，这些编辑器各有特点，基本使用方法是类似的：

  * 加载组件
  * 设置内容
  * 处理图片、视频等上传工作
  * 获取内容

本篇以常用的轻量级编辑器 WangEditor 举例，详细介绍富文本笔记的显示和编辑工作。

#### **WangEditor**

要了解一款编辑器，比较简单快捷的方法就是看它的界面布局和效果显示，下图是 WangEditor 编辑器的实际展现效果：

如图所示，它支持一些常用的效果，如：

  * 有序列表、无序列表
  * H1 到 H5、超链接、引用、代码块
  * 下划线、删除线、加粗、斜体、不同字体
  * 文字颜色、文字背景颜色、emoji 表情
  * 表格、图片、视频

这些操作基本上能满足平时使用富文本编辑的需求，官网地址：

> [http://www.wangeditor.com/](wangEditor.png)

Keller 云笔记中依然使用 npm 命令安装 WangEditor 组件：

    
    
    npm install wangeditor
    

### 业务流程

富文本笔记编辑效果如下：

对应的业务流程说明：

  1. 用户选择打开一个笔记
  2. 当选择的是一个富文本笔记时，在右侧展示该笔记的内容，同时右上角有一个编辑图标
  3. 用户点击编辑图标，加载富文本笔记编辑器
  4. 编辑过程中，用户可以点击右上角的保存图标，保存笔记

### 服务端代码实现

本篇涉及到的服务端代码，只有 NoteController 和 NoteService 两个类的改动。

  * NoteController：添加获取、保存笔记内容的接口
  * NoteService：处理获取、保存笔记内容的业务逻辑

#### **Controller**

在 NoteController.java 中添加 get 方法用以获取笔记内容：

    
    
            /**
         * 获取笔记详情
         * @param kellerUserId  从 JWT 中解析到的用户 ID
         * @param noteId    用户的笔记 ID
         * @return
         */
        @GetMapping
        public ResponseEntity get(Integer kellerUserId,Integer noteId){
            if(StringUtils.isEmpty(kellerUserId,noteId)){
                return Response.badRequest();
            }
            return Response.ok(service.get(kellerUserId,noteId));
        }
    

在 NoteController.java 中添加 save 方法用以保存笔记内容：

    
    
            /**
         * 保存笔记内容
         * @param kellerUserId  必填
         * @param noteId        必填
         * @param text          纯文本内容
         * @param html          Html 内容
         * @return
         */
        @PostMapping("/save")
        public ResponseEntity save(Integer kellerUserId,Integer noteId,String text,String html){
            if(StringUtils.isEmpty(kellerUserId,noteId) ){
                return Response.badRequest();
            }
            if(text == null){
                text = "";
            }
            if(html == null){
                html = "";
            }
            NoteInfo noteInfo = new NoteInfo(noteId);
            noteInfo.setUserId(kellerUserId);
            noteInfo.setText(text);
            noteInfo.setHtml(html);
            return Response.ok(service.save(noteInfo));
        }
    

在保存时要注意：

  * 笔记内容是可以为空的，但是我们在通用 Mapper 中的设置是 null 值不更新，因此，要判断当 text、html 参数为 null 时，将其转换为空字符串。

#### **Service**

NoteService.jave 也是添加相应方法即可，没有特殊的处理逻辑，如下：

    
    
    public ResultData get(int userId,int noteId){
            NoteInfo noteInfo = getByUserIdAndNoteId(userId, noteId);
            if(noteInfo == null){
                return ResultData.error("笔记本不存在");
            }
            return ResultData.success(noteInfo);
        }
    

### 前端代码实现

前端主要涉及到的代码结构如下：

  * components
    * Note.vue：自定义组件，处理笔记详情
    * WangEditor.vue：自定义组件，封装 WangEditor 组件，用于编辑富文本内容

#### **定义接口**

首先，要在 api.js 中定义好本篇所需的两个接口：获取笔记内容、设置笔记内容：

    
    
    /**
     * 获取笔记内容 4006
     */
    export const req_getNoteInfoById = (noteId) => { 
        return axios.get(api, {
            params:{
                noteId:noteId
            },
            headers:{
                'method':'note',
                'token' : window.localStorage.getItem("token")
            }
        }).then(res => res.data).catch(err => err);  
    };
    
    /**
     * 设置笔记内容 4007
     */
    export const req_setNoteContent = (noteId,text,html) => { 
        return axios.post(api, {
            noteId:noteId,
            text:text,
            html:html
        },{
            headers:{
                'method':'note/save',
                'token' : window.localStorage.getItem("token")
            }
        }).then(res => res.data).catch(err => err); 
    };
    

#### **自定义 WangEditor 组件**

定义好接口后，我们来自定义一个 WangEditor.vue，用于按照项目中使用的场景处理 WangEditor。看到这里，大家获取会提出疑问：

  1. WangEditor 本身使用就很简单了，为什么还要自己再封装一下呢？
  2. 在“Keller 云笔记”中这样封装，在其他项目中还需要封装吗？
  3. 我使用其他的编辑器，也需要进行封装吗？

对此，根据我个人在项目中实际使用情况做如下解释：

  * WangEditor 使用非常简单，只需要定义一好要挂载的 HTML 节点，并做简单配置即可，短短十几行代码就能搞定。但是在项目中，用到编辑器的地方很可能不止一处，用到一次，就写一次同样的十几行吗？后期如果配置需要修改（如：图片上传的地址），要找到每一处使用编辑器的地方修改吗？因此，当我们不想当一名 CV 程序员时，就要学会封装。
  * 封装的思想，在任何项目都适用。
  * 封装的思想，在任何编辑器中都适用。

新建 WangEditor.vue 代码如下：

    
    
    <template>
        <div>
        <!-- 挂载编辑器菜单栏的 Div -->
            <div id="editor1"></div>
        <!-- 挂载编辑器内容编辑区的 Div -->
            <div class="content" id="editor2">
            </div>
        </div>
    </template>
    <script>
        import Editor from 'wangeditor';
        export default {
            data() {
                return {
                    editor: new Editor("#editor1","#editor2")
                }
            },
            methods: {
            },
            mounted() {
            }
        }
    </script>
    

以上几行代码引入 WangEditor 组件，并将其挂载在页面中，要注意的是：使用 WangEditor 组件可以将菜单栏和内容编辑区挂载在一个 div
上，也可以将菜单栏和内容编辑区分别挂载在两个 div 上。在此建议使用后一种方法，因为前者会使用默认的 400PX
的高度，比较难以调整大小和样式；后者的方法可以灵活调节其样式。

WangEditor 组件的初始化方法写在 mounted() 钩子方法中比较合适，重点需要注意的方法如下：

    
    
    mounted() {
                let editor = this.editor;
              // 配置上传文件时的参数名
                editor.customConfig.uploadFileName = "file";
              // 配置上传图片的服务端地址
                editor.customConfig.uploadImgServer = editorImgUploadUrl;
              // 配置上传图片时带的自定义参数
                editor.customConfig.uploadImgParams = {
                    token: window.localStorage.getItem("token")
                };
                // 配置上传图片的回调方法
                editor.customConfig.uploadImgHooks = {
                    customInsert: function(insertImg, result, editor) {
                        // 图片上传并返回结果，自定义插入图片的事件（而不是编辑器自动插入图片！！！）
                        // insertImg 是插入图片的函数，editor 是编辑器对象，result 是服务器端返回的结果
                        // result 必须是一个 JSON 格式字符串！！！否则报错
                        // 举例：假如上传图片成功后，服务器端返回的是 {url:'....'} 这种格式，即可这样插入图片：
                        var url = result.data;
                        insertImg(url);
                    }
                };
              // 创建编辑器
                this.editor.create();
            }
    

WangEditor 内容编辑区需要设置的内容为 HTML 代码，可以获取到的内容为纯文本格式的字符串和 HTML 格式的字符串两种，因此，为
WangEditor.vue 添加三个可以调用的方法，供父组件调用：

  * load(html)：加载 WangEditor.vue 作为子组件时调用的设置内容的方法
  * getText()：获取编辑器内容编辑区的纯文本字符串
  * getHtml()：获取编辑器内容编辑区的 HTML 字符串

    
    
                getText(){
                    return this.editor.txt.text();
                },
                getHtml(){
                    return this.editor.txt.html();
                },
                load(html){
                    this.editor.txt.html(html);
                }
    

至此，自定义的 WangEditor 组件配置完毕，可以用它愉快地编辑富文本笔记了。

#### **自定义 Note 组件**

WangEditor 组件封装完毕后，我们还需要一个组件用于显示笔记内容。该组件的页面布局如下：

主要实现如下功能：

  * 默认为预览状态，显示笔记本标题及笔记内容
  * 点击编辑图标，进入编辑状态，按照笔记类型加载相应的编辑器
  * 在编辑状态点击保存按钮，保存修改的内容
  * 在编辑状态点击预览按钮，保存修改的内容，并切换到预览状态，显示编辑好的笔记内容

新建 Note.vue，引入 WangEditor.vue 组件：

    
    
        import WangEditor from "./WangEditor.vue"
        export default {
            components: {
                WangEditor,MarkDown
            }
      }
    

该页面所需的数据为：

  * editMode：模式切换开关，用于控制编辑模式和预览模式的相互切换
  * note：笔记对象，用于展示笔记详情

    
    
    data() {
                return {
                    //编辑模式
                    editMode: false,
                    note: {}
                }
            },
    

Note.vue 页面布局如下：

    
    
    <!-- 笔记详情 -->
    <template>
        <div>
            <el-row class="border-2-info font18 ColorCommon marginBottom10 padding5">
                <el-col :span="20">
                    <div class="font-bold ">{{note.title}}</div>
                </el-col>
                <!-- 操作按钮 -->
                <el-col :span="2" class="alignRight" v-show="!editMode">
                    <!-- 编辑 -->
                    <el-link :underline="false" @click="handleEdit()">
                        <i class="el-icon-edit font24" title="编辑"></i>
                    </el-link>
                </el-col>
                <!-- 操作按钮 -->
                <el-col :span="2" class="alignRight" v-show="editMode">
                    <!-- 预览 -->
                    <el-link :underline="false" @click="handleView()">
                        <i class="el-icon-view font24" title="预览"></i>
                    </el-link>
                </el-col>
                <!-- 操作按钮 -->
                <el-col :span="2" class="alignRight" v-show="editMode">
                    <!-- 保存 -->
                    <el-link :underline="false" @click="handleSave()" v-show="editMode">
                        <i class="el-icon-check font24" title="保存"></i>
                    </el-link>
                </el-col>
    
            </el-row>
            <!-- 仅在富文本笔记的编辑模式中使用 WandEditor -->
            <WangEditor v-show="editMode && note.type == 0" ref="wangEditor"></WangEditor>
            <!-- 阅读模式 -->
            <div v-show="!editMode" class="ColorMain" v-html="note.html"></div>
        </div>
    </template>
    

Note.vue 中重要的方法有：

init()：当选中不同的笔记后，Note.vue 会加载不同的内容，为了在笔记切换过程中造成的数据混乱，要在每次加载时初始化 Note.vue
使用的数据。

    
    
                //初始化数据，防止在笔记切换过程中造成的数据混乱
                init() {
                    this.editMode = false;
                    this.note = {};
                }
    

Note.vue 将作为 Home.vue 的子组件，因此要提供一个 load() 方法供父组件在合适的时机调用。

    
    
                //加载数据之前，先初始化原有数据
                load(note) {
                    this.init();
                    this.note = note;
                    this.getNoteInfo();
                }
    

handleSave()：保存笔记内容，调用子组件的方法分别读取纯文本内容和 HTML
内容，同时判断是否其和笔记原本内容相比有无变化，仅在发生变化时才调用服务端接口进行更新。

    
    
    handleSave() {
                    let newText;
                    let newHtml;
                    if(this.note.type == 0){
                        newText = this.$refs.wangEditor.getText();
                        newHtml = this.$refs.wangEditor.getHtml();
                    }else{
                        return;
                    }
                    //如果笔记内容有变化，保存
                    if(this.note.html !== newHtml){
                        req_setNoteContent(this.note.id, newText,newHtml).then(response => {
                            let {
                                success,
                                message
                            } = response;
                            //应答不成功，提示错误信息
                            if (success !== 0) {
                                this.$message({
                                    message: message,
                                    type: 'error'
                                });
                            } else {
                                this.$notify({
                                    title: '保存成功',
                                    type: 'success'
                                });
                                this.note.text = newText;
                                this.note.html = newHtml;
                            }
                        });
                    }else{
                    //笔记内容无变化，直接提示保存成功
                        this.$notify({
                            title: '保存成功',
                            type: 'success'
                        });
                    }
    
                }
    

#### **在用户主页中使用 Note 组件**

定义好 Note.vue 后，就可以在 Home.vue 中使用了：

    
    
            // 引入 Note.vue
            import Note from "../components/Note.vue"
            // 将 Note.vue 声明为子组件
            components: {
                NotesList,NoteList,Note
            },
    

在页面相应位置使用 Note 组件：

    
    
                  <!-- 右侧笔记内容 -->
                    <el-main class="note-info">
                        <Note ref="note"></Note>
                    </el-main>
    

在选中笔记列表中的笔记后，调用 Note 组件，加载笔记内容：

    
    
    getNote(note) {
            this.$refs.note.load(note);
    },
    

### 效果测试

选中笔记后，显示笔记内容：

点击编辑图标后，进入编辑状态，加载富文本编辑器：

点击预览按钮，保存笔记，并切换到预览状态

### 源码地址

本篇完整的源码地址：

> <https://github.com/tianlanlandelan/KellerNotes>

### 小结

本篇带领大家使用 WangEditor
完成了富文本笔记的预览和编辑功能，在源码中包含上传图片、显示图片等完整功能。网上有很多不同的富文本编辑器，在闲暇之余，大家可以体验一下其他编辑器，在使用方法和需要配置的参数上都是类似的，如果有更好的体验，可以相互推荐一下。

