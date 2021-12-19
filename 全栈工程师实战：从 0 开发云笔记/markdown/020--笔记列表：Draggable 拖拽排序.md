### 前言

通过上一篇的学习，大家对于 Vue 组件化有了一定的了解。本篇继续使用组件化的思想来实现笔记列表，同时在组件化的基础上，对笔记列表实现拖拽排序的功能。

### 知识准备

学习之前，先来了解一下 Vue 中完成拖拽排序功能的组件：vue.Draggable。

#### **vue.Draggable**

官网是这样介绍该组件的：

> Vue component (Vue.js 2.0) or directive (Vue.js 1.0) allowing drag-and-drop
> and synchronization with view model array. -- www.npmjs.com

也就是说该组件支持拖放操作，并且可以同步视图模型数组数据。 Draggable 基于 Sortable.js，主要特性有：

  * 支持拖拽、选择文本、智能滚动
  * 支持触摸设备
  * 支持不同列表之间的拖拽操作

参考文档地址：

> <https://www.npmjs.com/package/vuedraggable>

在 Keller 云笔记项目中，使用 `npm install --save vuedraggable` 命令即可安装，如图：

### 业务流程

本篇要实现的效果如下：

本篇涉及到的业务流程说明：

  * 展示笔记列表：
    * 进入用户主页后，获取到笔记本列表
    * 默认显示笔记本列表中第一个笔记本中的笔记
    * 当选中不同的笔记本后，显示被选中的笔记本中的笔记
  * 修改笔记排序：
    * 笔记列表可以进行拖拽操作
    * 当拖拽时，根据笔记当前在列表中的排序修改笔记的顺序

### 服务端代码实现

本篇在服务端没有新的知识点，只是业务逻辑的开发，在此仅以 4001 新建笔记接口举例，代码结构如下：

  * entity
    * NoteInfo.java：笔记实体类
  * mapper
    * NoteMapper.java：笔记数据库映射类
  * service
    * NoteService.java：笔记业务逻辑类
  * controller
    * NoteController.java：笔记接口处理类

#### **entity**

新建 NoteInfo.java 类，按照数据库设计中的 note_info 表中数据格式设计字段，特别要注意的是笔记内容字段。

content、contentMD 字段的长度按照需要进行设计：

  * 一般可以设计长度为 65535 以内，在创建数据表时将自动映射为 Text 类型，可以存储最大 64Kb 的内容；
  * 如果笔记内容可能很长，可以将长度设置超过 65535，在创建数据表时将自动映射为 MediumText 格式，可以存储最大 16Mb 的内容；
  * 更大的还有 LongText 类型，可以存储最大 4Gb 的内容。一下读取 4Gb 的数据，服务器也吃不消，网络也吃不消，因此不建议使用。

content、contentMD 字段要标注为明细字段：

  * 将字段标记为 `detailed = true` 时，表示该字段在查询列表时不会返回；
  * 因为笔记内容会很长，如果获取笔记列表时返回每个笔记的内容，HTTP 的传输量会特别大，也比较耗时。

参考代码如下：

    
    
    @TableAttribute(name = "note_info",comment = "笔记详情表")
    public class NoteInfo extends BaseEntity {
    
        @FieldAttribute
        @KeyAttribute(autoIncr = true)
        private Integer id;
    
        ... ...
    
        @FieldAttribute("笔记类型 0:富文本笔记 1:MarkDown笔记")
        private Integer type;
    
        @FieldAttribute("笔记标题")
        private String title;
    
        @FieldAttribute(value = "笔记内容",length = 70000,detailed = true)
        private String content;
    
        @FieldAttribute(value = "MarkDown 笔记内容",length = 70000,detailed = true)
        private String contentMD;
    
        @FieldAttribute("排序")
        @SortAttribute
        private int sort;
    
            ... ...
    }
    

#### **mapper**

新建 NoteMapper.java 继承 BaseMapper，实例化为 NoteInfo 类型：

    
    
    @Mapper
    public interface NoteMapper extends BaseMapper<NoteInfo> {
    }
    

#### **test**

使用 NoteMaper 继承的通用 Mapper 方法，创建 note_info 表：

    
    
            @Resource
        private NoteMapper noteMapper;
        @Test
        public void createNoteTable(){
            noteMapper.baseCreate(new NoteInfo());
        }
    

#### **service**

新建 NoteService.java，这里需要注入 NotesService 组件，用以在每次新建笔记时，校验笔记本，并在新建完成后将笔记本中的笔记数量加
1：

  * 首先校验指定的笔记本是否是该用户的，避免非法访问篡改数据的情景；
  * 如果不是该用户的，返回笔记本不存在；
  * 否则，新建笔记；
  * 笔记创建成功后，将笔记所属的笔记本中笔记数量加 1。 

代码如下：

    
    
    @Service
    public class NoteService {
    
        @Resource
        private NoteMapper mapper;
    
        @Resource
        private NotesService notesService;
      public ResultData createNote(NoteInfo noteInfo){
                // 校验用户的笔记本
            NotesInfo notesInfo = notesService.getByIdAndUserId(noteInfo.getNotesId(),noteInfo.getUserId());
            if(notesInfo == null){
                return ResultData.error("笔记本不存在");
            }
    
                //添加笔记
            Integer result = mapper.baseInsertAndReturnKey(noteInfo);
            if(result == 1){
                  // 将笔记本中的笔记数量加 1
                notesService.addNoteCount(notesInfo);
                return ResultData.success(noteInfo);
            }else {
                return ResultData.error("创建笔记本失败");
            }
        }
    }
    

#### **controller**

新建 NoteController.java，按照接口文档，实现笔记相关的接口：

    
    
    @RestController
    @RequestMapping("/note")
    public class NoteController {
    
        @Resource
        private NoteService service;
    
          @PostMapping
        public ResponseEntity create(Integer kellerUserId,Integer notesId,String title,Integer type){
              //校验必填参数
            if(StringUtils.isEmpty(kellerUserId,notesId,title)){
                return Response.badRequest();
            }
              //如果没有指定笔记类型，默认为创建富文本笔记
            if(type == null){
                type = PublicConstant.NOTE_TYPE_RICH_TEXT;
            }else{
                  //校验笔记类型是富文本或者 MarkDown
                if(type != PublicConstant.NOTE_TYPE_RICH_TEXT && type != PublicConstant.NOTE_TYPE_MARK_DOWN){
                    return Response.badRequest();
                }
            }
            NoteInfo noteInfo = new NoteInfo(kellerUserId,notesId,title);
            noteInfo.setType(type);
            return Response.ok(service.createNote(noteInfo));
        }
    }
    

至此，服务端代码实现完毕，在文末有完整源码。

### 前端代码实现

前端代码结构如下：

  * components
    * NoteList.vue：自定义 Vue 组件，用于展示笔记列表 
    * Home.vue：用户主页，引入 NoteList.vue 作为子组件

#### **定义接口**

首先，在 api.js 中按照接口文档，定义好笔记相关功能所需的接口：

    
    
    /**
     * 添加笔记 4001
     */
    export const req_addNote = (note) => { 
        return axios.post(api, {
            notesId:note.notesId,
            title:note.title,
            type:note.type
        },{
            headers:{
                'method':'note',
                'token' : window.localStorage.getItem("token")
            }
        }).then(res => res.data).catch(err => err); 
    };
    
    /**
     * 获取笔记列表 4003
     */
    export const req_getNoteList = (notesId) => { 
        return axios.get(api, {
            params:{
                notesId:notesId
            },
            headers:{
                'method':'note/list',
                'token' : window.localStorage.getItem("token")
            }
        }).then(res => res.data).catch(err => err);  
    };
    /**
     * 笔记重排序 4005
     */
    export const req_noteReSort = (noteId,sort) => { 
        return axios.post(api, {
            noteId:noteId,
            sort:sort
        },{
            headers:{
                'method':'note/reSort',
                'token' : window.localStorage.getItem("token")
            }
        }).then(res => res.data).catch(err => err); 
    };
    

#### **自定义笔记列表组件**

笔记列表组件的基本功能是展示用户笔记本中的笔记列表，效果如下：

新建 NoteList.vue，使用 v-for 实现列表的布局：

    
    
    <template>
        <div>
                <!-- 笔记列表 使用 v-for -->
                <div class="font16 ColorCommon border-three padding5" v-for="item in list" :key="item.id" @click="currentNote(item)">
                    <div :class="note.id == item.id ? 'note-bg':'cursorPointer '">
                        <el-row>
                            <el-col :span="2" :offset="1">
                                <i class="el-icon-document"></i>
                            </el-col>
                            <el-col :span="14">
                                {{item.title.length > 10 ?item.title.substring(0,10) + '...' : item.title}}
                            </el-col>
                            <el-col :span="6">
                                {{format.formatDate(item.createTime)}}
                            </el-col>
                        </el-row>
                    </div>
                </div>
        </div>
    </template>
    

在 JS 中，获取笔记列表：

    
    
    getNoteList(notesId) {
                    req_getNoteList(notesId).then(response => {
                        //解析接口应答的json串
                        let {
                            data,
                            message,
                            success
                        } = response;
                        //应答不成功，提示错误信息
                        if (success !== 0) {
                            this.$message({
                                message: message,
                                type: 'error'
                            });
                        } else {
                            if (data.length > 0) {
                                this.list = data;
                                this.note = this.list[0];
                            }
                        }
                    });
                },
    

这一部分和上一篇的 NotesList.vue 没有区别。

#### **父组件触发子组件加载事件**

同上一篇一样，在 Home.vue 中引入 NoteList.vue：

    
    
        import NoteList from "../components/NoteList.vue";
        ... ...
        export default {
            components: {
                NotesList,NoteList
            },
            data() {
          ... ...
        }
      }
    

在页面中使用，为其设置 ref 属性，并监听其 @func 回调函数：

    
    
    <!-- 中间笔记列表 -->
                    <el-aside width="21%" class="note-list">
                        <NoteList @func="getNote" ref="noteList"></NoteList>
                    </el-aside>
    

将主动调用 NoteList.vue 中 load 方法的时机选择在：当笔记本列表组件 NotesList.vue
中当前选中的笔记本发生变化时，也就是在监听到 NotesList.vue 的回调方法时，主动调用 NoteList.vue
的加载方法。这样就能保证，在选中一个笔记本后，加载它里面的笔记列表：

    
    
                /**
                 * 笔记本列表组件 NotesList 中当前选中的笔记本有变化时，通知父组件刷新当前笔记本
                 * @param {Object} notes
                 */
                getNotes(notes) {
                    this.currentNotes = notes;
                    //加载笔记本中的笔记列表
                    this.$refs.noteList.load(notes.id);
                },
    

完成后的效果如下。

当选中笔记本“小小账本”时，显示“小小账本”中的笔记列表：

当选中笔记本“生活随笔”时，显示“生活随笔”中的笔记列表：

#### **拖拽排序**

在 NoteList.vue 中，引入 vue.Draggable 组件：

    
    
    import draggable from 'vuedraggable';
    
    components: {draggable}
    

引入后，使用 `<draggable>` 标签包裹笔记本列表：

    
    
            <draggable group="note" v-model="list" @change="noteListChange()">
                <!-- 笔记列表 -->
                <div class="font16 ColorCommon border-three padding5" v-for="item in list" :key="item.id" @click="currentNote(item)">
                    ... ...
                    </div>
            </draggable>
    

使用说明：

  * group：将vue.Draggable 组件分组，不同组的 vue.Draggable 可以是父子关系也可以是兄弟关系，其支持不同组件间的数据进行拖拽。
  * v-model：绑定的数据模型，要求是一个数组。在这里绑定了笔记列表数组，拖拽的变化会引起数组元素顺序的变化。
  * @change：引起拖拽变化时绑定的处理方法。

在这里，将 @change 的处理方法绑定为 noteListChange：

  * 遍历笔记数组，判断每一项数据元素的顺序有没有发生变化
  * 如果没有变化，不做处理
  * 如果有变化，调用接口 4005，修改笔记顺序

参考代码如下：

    
    
    noteListChange() {
                    for (var i = 0; i < this.list.length; i++) {
                        //如果笔记的顺序没有发生变化，不做处理
                        if(this.list[i].sort == i ){
                            continue;
                        }
                        //当笔记顺序发生变化时，重新排序
                        req_noteReSort(this.list[i].id, i).then(response => {
                            //解析接口应答的json串
                            let {
                                message,
                                success
                            } = response;
                            //应答不成功，提示错误信息
                            if (success !== 0) {
                                this.$message({
                                    message: message,
                                    type: 'error'
                                });
                            }
                        });
                    }
                }
    

实现后的效果如图，可以通过拖拽改变笔记本下笔记的顺序：

### 源码地址

本篇完整的源码地址如下：

>
> [https://github.com/tianlanlandelan/KellerNotes/tree/master/20.%E7%AC%94%E8%AE%B0%E5%88%97%E8%A1%A8](https://github.com/tianlanlandelan/KellerNotes/tree/master/20.笔记列表)

### 小结

本篇实现了显示所选笔记本下的笔记列表功能，同时使用 vue.Draggable 组件实现了笔记列表的拖拽排序功能。

vue.Draggable 是支持不同列表之间的拖拽的，大家可以自己实现一下笔记本列表的拖拽排序，以及将笔记拖拽到不同的笔记本中。

