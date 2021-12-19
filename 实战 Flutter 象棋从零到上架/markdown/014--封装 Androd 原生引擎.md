到上了节为止，我们在 iOS 平台下封装了 eleeye 引擎，我们已经可以愉快地与 iOS 设备进行单机对战了。

Android 和 iOS 两个移动系统，对于 C++ 原生代码的混编支持都比较友好，而且在进行 iOS 封装前我们已经对 eleeye
代码进行了必要的改造，这些改造对于 Android 都是同样适用的。

## 本节概要

  * 修改 Android 包名
  * 添加原生的 Library 包装类
  * 配置 cmake 方式的 NDK 任务
  * 在 Method Channel 中控制引擎

## 修改 Android 包名

在开始进行 Android 对 eleeye 引擎的封装之前，我们先来对面另外一个不太相关的小问题：

Flutter 创建的项目，默认 android 包名都是 com.example.<项目名>
这样的，如果我们将一个产品发布到应用市场，这个包名很显然不是你想要的。我们得把它改成一个唯一的、合适的包名。

很明显，我认为合适的包名也不大概率不适合你，你应该规划自己的包含，我会带大家一步一步地修改包名：

第一步，确认自己要用的包名，比如我将使用 cn.apppk.books.flutterchess 这个包含名。

第二步，我们回到 vscode，全局查找 com.example.chessroad 这个字符串，全局替换为
cn.apppk.books.flutterchess。

第三步，使用 Android Studio 打开我们的项目的 Android 模块，首先选中 src/java，按 `Cmd+N`
选择「Package」，在弹出棋中输入你的包名。

第四步，将 MainActivity.java 拖拽到 flutterchess 包中，如下图所示：

![将 MainActivity.java 拖拽到 flutterchess
包](https://images.gitbook.cn/0xJnSg.png)

第五步，逐级删除掉空的 com.example.chessroad 包。

这样，包名就改过来了，我们开始吧进入这一节的正题吧！

## 添加原生 Library 包装类

我们先将 eleeye 引擎的代码链接到 android 项目文件夹下边。首先进入项目根目下的 android 目录：

    
    
    android git:(master) $ cd app/src/main
    main git:(master) $ mkdir cpp
    cpp git:(master) $ ln -s ~/chessroad/eleeye engine
    

根据 Flutter 一侧的接口要求，我们在 Android Studio 中添加引擎实现的 CChessEngine 类，将文件创建在与
MainActivity 相同的层级：

    
    
    public class CChessEngine {
    
        static {
            System.loadLibrary("engine");
        }
    
        public native int startup();
    
        public native int send(String arguments);
    
        public native String read();
    
        public native int shutdown();
    
        public native boolean isReady();
    
        public native boolean isThinking();
    }
    

接着，我们使用 javah 命令，为 CChessEngine 生成 JNI 头文件。打开命令行工具，进入
android/app/src/main/java 文件夹，执行以下命令：

    
    
    java git:(master) $ javah cn.apppk.books.flutterchess.CChessEngine
    

> 找不到 javah 什么的，表示你的 JDK/JRE 环境没见有配置好，这个百度一下一大把解决方案，不用在这里啰嗦了吧？

上边的命令执行完成后，将在 java 目录下生成一个 `cn_apppk_books_flutterchess_CChessEngine.h`
文件。这个头文件按 CChessEngine 中的方法签名，输出了实现文件 JNI 签名，稍后我们会用到它。

接着，我们在 android/app/src/main/cpp 目录下新建一个名字叫 cchess-engine.cpp
的文件，打开这个空的文件，将上一步生成的 .h 文件中的方法签名复制到 这个 cpp 文件中，并给每个方法加上空的方法体：

> 在 Android Studio 中，左中角可以切换资源组织方式，像 src 下的 cpp 文件夹可能在 Android 视图下看不到，切换到
> Project 视图就 OK 了。
    
    
    #include <jni.h>
    
    extern "C" {
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_startup(JNIEnv *, jobject) {
        return 0;
    }
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_send(JNIEnv *, jobject, jstring) {
        return 0;
    }
    
    JNIEXPORT jstring JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_read(JNIEnv *, jobject) {
        return NULL;
    }
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_shutdown(JNIEnv *, jobject) {
        return 0;
    }
    
    JNIEXPORT jboolean JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_isReady(JNIEnv *, jobject) {
        return false;
    }
    
    JNIEXPORT jboolean JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_isThinking(JNIEnv *, jobject) {
        return false;
    }
    
    }
    

到这里，前文创建的 `cn_apppk_books_flutterchess_CChessEngine.h` 文件的使命就完成了，你可以大方地将它删除掉。

## 配置 cmake 方式 NDK 任务

在我们继续向 cchess-engine.cpp 里面添加有效代码之前，我们要做一点 cmake 的配置工作了。

在 app 文件夹下，我们添加 CMakeLists.txt 文件，其内容如下：

    
    
    cmake_minimum_required(VERSION 3.4.1)
    
    add_library( # Sets the name of the library.
            engine
    
            # Sets the library as a shared library.
            SHARED
    
            # Provides a relative path to your source file(s).
            src/main/cpp/cchess-engine.cpp
            src/main/cpp/engine/command-channel.cpp
            src/main/cpp/engine/command-queue.cpp
            src/main/cpp/engine/book.cpp
            src/main/cpp/engine/eleeye.cpp
            src/main/cpp/engine/evaluate.cpp
            src/main/cpp/engine/genmoves.cpp
            src/main/cpp/engine/hash.cpp
            src/main/cpp/engine/movesort.cpp
            src/main/cpp/engine/position.cpp
            src/main/cpp/engine/preeval.cpp
            src/main/cpp/engine/pregen.cpp
            src/main/cpp/engine/search.cpp
            src/main/cpp/engine/ucci.cpp)
    
    
    find_library( # Sets the name of the path variable.
            log-lib
    
            # Specifies the name of the NDK library that
            # you want CMake to locate.
            log)
    
    
    target_link_libraries( # Specifies the target library.
            engine
    
            # Links the target library to the log library
            # included in the NDK.
            ${log-lib})
    

接着，我们在 app 下的 build.gradle 文件中，修改 android 代码块，在其尾部添加 cmake 的任务，修改为下边样子：

    
    
    android {
    
        ...
    
        externalNativeBuild {
            cmake {
                path "CMakeLists.txt"
            }
        }
    }
    

保存 build.gradle 文件，在 build.gradle 文件顶部弹出的提示条中点击「Sync Now」进行配置同步。

配置同步后，如果你细心点可以发现：CChessEngine 类里面的 native 方法已经与 cchess-engine.cpp
里面的方法绑定在一起的，按 Cmd 点击方法名相互可以切换。

> 在开发过程中，cpp 部分的代码也可以像 Java 代码一样，支持断点跟踪、调试，这对后续的调试很重要。

## 在 Method Channel 中控制引擎

我们现在可以参考 iOS 里面的引擎包裹逻辑，实现 cchess-engine.cpp 里面的各个方法。需要多做一点工作的是 startup
接口，这个接口的实现代码中将启动新的线程。

我们先定义两个全局变量：

    
    
    extern "C" {
    
    State state = Ready;
    pthread_t thread_id = 0;
    
    ...
    }
    

然后在 cchess-engine.cpp 的两个全局变量之后，定义一个线程函数，将 eleeye 引擎的入口包裹在以下方法中：

    
    
    void *engineThread(void *) {
    
        printf("Engine Think Thread enter.\n");
    
        engineMain();
    
        printf("Engine Think Thread exit.\n");
    
        return NULL;
    }
    

接着，我们在 startup 实现方法中使用此函数方法，在新线程中启动 eleeye 引擎的消息循环。

由于在 startup 方法中用到了后续才出现的 send 和 shutdown 方法，而 C 语言要求先声明后使用，所以我们还要将 send 和
shutdown 方法的声明语句放在 startup 方法之前：

    
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_send(JNIEnv *env, jobject, jstring command);
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_shutdown(JNIEnv *, jobject);
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_startup(JNIEnv *env, jobject obj) {
    
        if (thread_id) {
            Java_cn_apppk_books_flutterchess_CChessEngine_shutdown(env, obj);
            pthread_join(thread_id, NULL);
        }
    
        // getInstance() 有并发问题，这里首先主动建立实例，避免后续创建重复
        CommandChannel::getInstance();
    
        usleep(10);
    
        pthread_create(&thread_id, NULL, engineThread, NULL);
    
        Java_cn_apppk_books_flutterchess_CChessEngine_send(env, obj, env->NewStringUTF("ucci"));
    
        return 0;
    }
    

以下其它方法的处理方式与在 iOS 中的逻辑是一致的，完整的 cchess-engine.cpp 代码如下：

    
    
    #include <jni.h>
    #include <sys/types.h>
    #include <pthread.h>
    #include <unistd.h>
    #include <stdio.h>
    #include <string.h>
    #include "engine/eleeye.h"
    #include "engine/engine-state.h"
    #include "engine/command-channel.h"
    
    extern "C" {
    
    State state = Ready;
    pthread_t thread_id = 0;
    
    void *engineThread(void *) {
    
        printf("Engine Think Thread enter.\n");
    
        engineMain();
    
        printf("Engine Think Thread exit.\n");
    
        return NULL;
    }
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_send(JNIEnv *env, jobject, jstring command);
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_shutdown(JNIEnv *, jobject);
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_startup(JNIEnv *env, jobject obj) {
    
        if (thread_id) {
            Java_cn_apppk_books_flutterchess_CChessEngine_shutdown(env, obj);
            pthread_join(thread_id, NULL);
        }
    
        // getInstance() 有并发问题，这里首先主动建立实例，避免后续创建重复
        CommandChannel::getInstance();
    
        usleep(10);
    
        pthread_create(&thread_id, NULL, engineThread, NULL);
    
        Java_cn_apppk_books_flutterchess_CChessEngine_send(env, obj, env->NewStringUTF("ucci"));
    
        return 0;
    }
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_send(JNIEnv *env, jobject, jstring command) {
    
        const char *pCommand = env->GetStringUTFChars(command, JNI_FALSE);
    
        if (pCommand[0] == 'g' && pCommand[1] == 'o') state = Thinking;
    
        CommandChannel *channel = CommandChannel::getInstance();
    
        bool success = channel->pushCommand(pCommand);
        if (success) printf(">>> %s\n", pCommand);
    
        env->ReleaseStringUTFChars(command, pCommand);
    
    
        return success ? 0 : -1;
    }
    
    JNIEXPORT jstring JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_read(JNIEnv *env, jobject) {
    
        char line[4096] = {0};
    
        CommandChannel *channel = CommandChannel::getInstance();
        bool got_response = channel->popupResponse(line);
    
        if (!got_response) return NULL;
    
        printf("<<< %s\n", line);
    
          // 收到以下响应之一，标志引擎退出思考状态
        if (strstr(line, "readyok") ||
            strstr(line, "ucciok") ||
            strstr(line, "bestmove") ||
            strstr(line, "nobestmove")) {
    
            state = Ready;
        }
    
        return env->NewStringUTF(line);
    }
    
    JNIEXPORT jint JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_shutdown(JNIEnv *env, jobject obj) {
    
        Java_cn_apppk_books_flutterchess_CChessEngine_send(env, obj, env->NewStringUTF("quit"));
    
        pthread_join(thread_id, NULL);
    
        thread_id = 0;
    
        return 0;
    }
    
    JNIEXPORT jboolean JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_isReady(JNIEnv *, jobject) {
        return static_cast<jboolean>(state == Ready);
    }
    
    JNIEXPORT jboolean JNICALL
    Java_cn_apppk_books_flutterchess_CChessEngine_isThinking(JNIEnv *, jobject) {
        return static_cast<jboolean>(state == Thinking);
    }
    
    }
    

离完成 Android 引擎还差一道工序，就是在我们的 MainActivity 类中注册 Method Channel，并调用我们的
CChessEngine 类中的原生实现。

由于这儿只是将来自 Flutter 的方法调用转交给 Native 实现，所以这块的代码并没有什么复杂的逻辑：

    
    
    public class MainActivity extends FlutterActivity {
    
        private static final String ENGINE_CHANNEL = "cn.apppk.chessroad/engine";
    
        private CChessEngine engine;
    
        @Override
        public void onCreate(Bundle savedInstanceState) {
    
            super.onCreate(savedInstanceState);
    
            engine = new CChessEngine();
    
            FlutterEngine fe = getFlutterEngine();
            if (fe == null) return;
    
            MethodChannel methodChannel =
              new MethodChannel(fe.getDartExecutor(), ENGINE_CHANNEL);
    
            // 对接 Flutter 侧对 Mehtod Channel 的调用
            methodChannel.setMethodCallHandler((call, result) -> {
    
                    switch (call.method) {
    
                      case "startup":
                        result.success(engine.startup());
                        break;
    
                      case "send":
                        result.success(engine.send(call.arguments.toString()));
                        break;
    
                      case "read":
                        result.success(engine.read());
                        break;
    
                      case "shutdown":
                        result.success(engine.shutdown());
                        break;
    
                      case "isReady":
                        result.success(engine.isReady());
                        break;
    
                      case "isThinking":
                        result.success(engine.isThinking());
                        break;
    
                      default:
                        result.notImplemented();
                        break;
                    }
                }
            );
        }
    
        @Override
        public void configureFlutterEngine(@NonNull FlutterEngine flutterEngine) {
    
            GeneratedPluginRegistrant.registerWith(flutterEngine);
        }
    }
    

编译，运行，收工！

多体验一下我们的产品，试着按自己的思路去改进它。

请记得将代码提交到 git 仓库！

## 本节回顾

基于前一节的封装 iOS 原生引擎的经，在 Android 一侧封装 eleeye 原生引擎的流程大同小异。

在本节中，我们使用了 JNI 方式来将 eleeye 引擎进行封装。由于 jni 的函数名中包含了项目的包名，所以在开始规划 jni
的函数结构前，我们需要确认项目的包名。

> 对于 Android 程序来说，包名就是产品的 ID，必须是唯一且有意义的。Flutter 环境为了简化项目的创建过程，一般都给 android
> 配置了像「com.example.xxx」这样的包名。这样的包保无法保证唯一，必须在发布到产品环境前进行修改。

这之后，我们在 Java 中添加了对 JNI 库的包装类，Method Channel 将通过此类中的 native 方法与原生引擎对话。

随后，我们导入了 eleeye 的源码，并配置了使用 cmake 方式来管理 NDK 任务的脚本。

最后，我们在 Method Channel 中完成了对引擎的逻辑控制。

## 延展信息

Android 和 iOS 中使用了同一份 eleeye 的经过我们修改的代码，共用一份代码更利于维护和除错。

Xcode 的编译环境要求将代码放置在项目的目录之下，而我们希望将代码放在一个公共的位置，由 Android 和 iOS
共同访问。解决这个问题办法是文件链接：

在 macOS 上，使用 `ln -s <origin-file-path> <target-file-path>` 命令即可容易地解决此问题。

