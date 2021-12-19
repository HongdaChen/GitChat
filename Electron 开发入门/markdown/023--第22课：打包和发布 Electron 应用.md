到目前为止，我们已经学习了很多 Electron 基础知识，不过还有一个问题没解决，就是我们开发了一款基于 Electron
的应用，自己和团队用肯定是没有任何问题，因为团队每个成员的机器上都有 Node.js 和 Electron
环境，但如何将这款应用分发给用户呢？除了为自己团队开发的应用外，都会面临这个问题。

肯定不能要求用户在安装应用之前，先安装 Node.js、Electron
以及其他必要的依赖库，因为用户有可能是初学者，他们希望能通过学习本课程的内容来完成安装等。

所以解决这个问题的唯一办法就是我们将所需的一切软件和资源都打包，到时只要分发给用户一个安装程序（通常是一个可执行文件），用户只需要双击运行安装程序，然后按
step by step 的方式就可以搞定一切。但问题又来了，尽管用 Node.js + Electron
的方式开发应用相当快捷，但也有缺点，就是需要依赖于大量的模块（包括官方的和第三方的，也可以称这种情况为依赖地狱），因此需要将这些模块和其他资源都打包在一个安装程序中发布。如果自己完成一些，用户是方便了，但程序员就麻烦了，不过可以使用一些第三方的工具自动打包
Electron 应用。

在这里会介绍如下三款开源免费的打包工具，Electron 打包应用有很多，本课主要介绍 electron-
packager，其他类似的打包工具的使用方法差不多，通常都支持命令行方式和 API 方式。

### 22.1 安装 electron-packager

electron-packager 是一款非常强大的 Electron 应用打包工具，是开源免费的，读者可以到 GitHub 下载 electron-
packager 的源代码，以及查看 electron-packager 的官方文档。

[Electron-packager 官网详见这里。](https://github.com/electron-userland/electron-
packager)

在使用 electron-packager 之前，首先要安装 electron-packager，读者可以使用下面的命令将 electron-
packager 安装到当前工程中。

    
    
    npm install electron-packager --save-dev
    

或者使用下面的命令全局安装 electron-packager。

    
    
    npm install electron-packager -g
    

安装完 electron-packager 后，直接在终端中输入 electron-packager 命令，就会列出 electron-packager
工具的所有命令行参数及详细解释。

electron-packager 支持如下两种使用方式：

  * 命令行
  * API

下面将详细解释这两种使用方式。

### 22.2 electron-packager 的基本用法

本节课将介绍 electron-packager 基本的命令行使用方式。现在准备一个 Electron 工程（本例的工程目录是
release/Test），首先使用“electron .”命令运行工程，效果如下图所示。

![](https://images.gitbook.cn/FqvpLkYuPMyTO5_uXvEKIHOCNXqh)

接下来在终端进入 release/Test 目录，然后输入下面的命令打包 Test 应用。

    
    
    electron-packager . --electron-version=3.0.2
    

如果执行成功，在不同类型的操作系统中会生成不同的目录，这里只介绍 Mac OS X 和 Windows。

在 Mac OS X 下，会生成 Test-darwin-x64 目录，在 Windows 下，会生成 Test-win32-x64
目录，这两个目录中分别包含了 Mac OS X 和 Windows 版本的安装包文件。其中 Test-darwin-x64
目录中包含了如下图的一些文件，Test（Test.app）就是 Mac OS X 版本的运行程序，这是 Mac OS X
下特有的目录，里面包含了与应用程序相关的资源。

![image.png](https://images.gitbook.cn/Ft27OLP-kYWRn-aYFkFCKnAqmCbr)

在 Windows 下，Test-win32-x64 目录下的文件就多了，大多数都是 dll 文件。在 Test-win32-x64 目录中包含了一个
Test.exe 文件，该文件就是 Test 应用的可执行文件，双击即可运行。

下面解释一下前面的打包命令，其中 electron-packager 命令后面的点（.）表示要打包当前目录的工程，后面的 --electron-
version 命令行参数表示要打包的 Electron 版本号，注意，这个版本号不是本地安装的 electron 版本号，而是打包到安装包中的
electron 版本，但建议打包的 Electron 版本尽量与开发中使用的 Electron 版本相同，否则容易出现代码不兼容的问题。

在打包的过程中，electron-packager 会下载指定的 Electron 安装包，下载过程如下图所示。

![image.png](https://images.gitbook.cn/FiceEBM0x8qgiluFPTFk7INlVZXQ)

下载完后，就会进行打包，并生成前面描述的目录。

现在运行打包后的程序，Mac OS X 的效果如下图所示。

![](https://images.gitbook.cn/FkuyUD19dsOgN2YJBSwCpZsTdFv5)

Windows 下的效果如下图所示。

![](https://images.gitbook.cn/Fqyi8mAIEW7Nces1qG0KeNsyBYnE)

不管是 Mac OS X，还是 Windows，页面上方的图像都没显示出来，图像与工程的目录关系如下图所示。

![](https://images.gitbook.cn/Fj0YkLCKoJy-LFQCJWUIRUZ5GpHx)

而在 index.html 文件中使用下面的代码引用图像。

    
    
    <img  src="../../../images/ether.png"/>
    

那么这个图像没出来，应该如何处理呢？

### 22.3 如何处理应用中的资源

继续前面的例子，由于图像目录没在工程目录中，因而 electron-packager 并没有将图像目录进行打包，为了解决这个问题，可以使用如下 4 种方法。

#### 方法1：亡羊补牢（复制图像目录到包目录）

由于 Mac OS X 和 Windows 版本的应用打完包后都是一个目录，因而可以在后期将 images 目录复制到包目录的某个地方。对于本例来说，在
Mac OS X 下，需要将 images 目录放在 index.html 文件往上 3 层的目录，也就是 Test.app 目录中。

在 Windows 中，当前目录并不是从 Test.exe 开始算的，而是从 `<打包目录>\resources\app` 目录开始算的。在 App
目录中包含了如下 4 个文件，也就是工程中的 4 个文件。

![avatar](https://images.gitbook.cn/FoneFbehEYLVxD94l5B5pMP3l3er)

读者可以直接在浏览器中浏览 index.html 页面，如果图像路径是正确的，会看到如下图所示的效果。

![](https://images.gitbook.cn/Fp2B0mfiylEJdepSYaLvtWsBMmWq)

问题是现在图像路径不正确，因此需要将 images 目录放到与打包目录同一级的目录中，这样图像就可以出来了。

#### 方法2：直接在包目录中修改图像路径

electron-packager 命令在打包时，会将 Electron
应用的源代码也放到包目录中，因此可以直接修改包目录中的源代码，将图像或其他资源的路径指向正确的文件。

在 Mac OS X 中，打开 Test.app/Contents/Resources/app 目录，会看到上述 4 个文件，直接修改 index.html
页面的代码即可。

在 Windows 下，打开 `<包目录>\resources\app` 目录，在该目录中包含了上述 4 个文件，直接修改 index.html
页面的代码即可。

#### 方法3：在开始时将资源放到工程目录中

每次打包后都需要手工修改太麻烦，因此最好的方式就是在开发时就将资源路径搞定，通常需要将资源文件或目录放到工程目录中，如下图所示。

![](https://images.gitbook.cn/FiUnl-SG6ZS67ZRrQEN-82L85ZSr)

这样就需要修改 index.html 页面中图像的路径，代码如下：

    
    
    <img  src="images/ether.png"/>
    

现在打包，再运行应用程序，图像就出来了。

#### 方法4：使用 Web 资源

这种方式非常容易理解，就是不使用本地资源，所有的资源都直接从网上获取，这样做的好处是不会出现资源异常的情况（服务器出问题除外），但缺点也很明显，就是资源从网络下载，速度比较慢，容易出现卡顿现象。如果使用的是局域网，那么情况会好很多。

打包时，应用中的其他资源也可以用上述 4 种方法处理。

### 22.4 打包任意工程目录

electron-packager 命令可以在任意目录执行，但要通过命令行参数指定工程目录的相对或绝对路径。

下面的命令指定了 Test 工程的相对目录，electron-packager 会在当前目录生成包目录。

    
    
    electron-packager Test  --electron-version=3.0.2
    

同样可以指定工程目录的绝对路径。

    
    
    electron-packager D:\MyStudio\resources\electron\src\release\Test  --electron-version=3.0.2
    

执行上面的命令后，electron-packager 命令在哪个目录下执行，就会在哪个目录下生成打包目录。

### 22.5 修改可执行文件的名称

生成的包目录中有一个可执行文件，对于 Mac OS X 来说，直接双击安装包，也就是 Test.app，然后运行，这是 Mac OS X
运行程序的方式，不过在 Test.app 目录内部有一个可执行文件，实际上双击 Test.app
后，是通过运行这个可执行文件来运行程序的。这个可执行程序在如下的目录：

    
    
    Test.app/Contents/MacOS
    

该目录中有一个 Test 文件，没有扩展名，这个文件其实是一个命令行程序，内部调用了 GUI API 创建窗口。运行
Test，首先会显示一个命令行窗口，然后会启动前面页面的窗口页面。

这个可执行文件（Test）的名称是可以修改的，但不能直接修改文件名，需要用 --executable-name 命令行参数修改可执行文件的名字。

    
    
    electron-packager . --executable-name new --electron-version=3.0.2
    

执行上面的命令后，会将可执行程序的名字改为 new。

在 Windows 下，可执行文件的名字就是 Test.exe，该文件名可以直接修改，当然，也可以使用下面的命令修改。

    
    
    electron-packager . --executable-name my --electron-version=3.0.2
    

执行这行命令后，就会在 Windows 版的包目录中生成 my.exe 文件。

> 如果包目录已经存在，可以使用 --overwrite 命令行参数覆盖包目录。

### 22.6 修改应用程序名称

Electron 工程必须包含一个 package.json 文件，该文件有一个重要的属性 name，那就是 Electron
工程的名字，有很多地方（尤其是 Mac OS X）都依赖于这个名字。例如，使用前面介绍的命令打包 Electron 应用时，生成的包目录名（Test-
darwin-x64 或 Test-win32-x64）的前缀（Test），第一个应用菜单的文本（如下图）都会使用这个 name 属性的值。

![](https://images.gitbook.cn/FvGUgGKGG3bM2WE6OWnQG245gLms)

如果想修改应用程序名，也不需要修改 package.json 文件中的 name 属性值，只需要通过 electron-packager
命令指定应用程序名即可，命令如下：

    
    
    electron-packager . hello  --electron-version=3.0.2
    

执行上面的命令后，会在当前目录生成名为 hello-darwin-x64 的目录，里面有一个 hello.app
目录，双击即可执行程序，效果如下图所示，很明显，菜单文本从 Test 变成了 hello。

![](https://images.gitbook.cn/FtXA_19PcntXejOJtuqGhDGN-vAU)

除了菜单文本和包目录名改变了，里面的可执行程序也从 Test 变成了 hello。

> 通过 electron-packager 命令指定的应用名称并不会修改 package.json 文件的 name 属性值，在 Mac OS X
> 系统中是通过 Info.plist 文件的相关属性描述的。electron-packager 命令修改的是 Info.plist 文件的属性值，在
> Windows 中应用名称只影响可执行文件名和包目录名。

### 22.7 修改应用程序图标

在默认情况下，生成的应用程序图标使用的是 Electron 的 Logo，如下图所示。

![image.png](https://images.gitbook.cn/FhDpiEn5w1Lc_zUkgnbXY6RjUCLz)

不过在实际的应用程序中通常会使用自己公司的 Logo 或为应用程序专门设计的 Logo 作为应用程序的图标，这就需要修改 Electron 应用的默认图标。

Mac OS X 版的 Electron 应用图标修改方式有多种，在 Test.app/Contents/Resources 目录中有一个
electron.icns 文件，该文件中包含了多个不同尺寸、但图案相同的图标，这是为了在 Mac OS X 不同的场景显示图标，直接替换
electron.icns 文件就可以改大部分 Electron 应用的图标。不过这里带来一个问题，electron.icns
文件是什么呢？该文件是做什么的呢？

其实 icns 文件是 Mac OS X 特有的图标集合文件，该文件中包含了多个不同尺寸的 png 图像，打开 electron.icns
文件，会看到如下图所示的效果。

![](https://images.gitbook.cn/FkZUg7edfIXxsax7TlHjtqgdcF6D)

icns 文件中的 png 图像尺寸包括 16 × 16、32 × 32、64 × 64、128 × 128、256 × 256 和 512 × 512
这六种尺寸，这些尺寸对应的文件名以及 PPI（屏幕密度，也就是每英寸包含的像素点个数）如下表所示。

文件名 | 尺寸 | 屏幕密度（PPI）  
---|---|---  
icon_16 × 16.png | 16 × 16 | 72  
icon_16 × 16@2x.png | 32 × 32 | 144  
icon_32 × 32.png | 32 × 32 | 72  
icon_32 × 32@2x.png | 64 × 64 | 144  
icon_128 × 128.png | 128 × 128 | 72  
icon_128 × 128@2x.png | 256 × 256 | 144  
icon_256 × 256.png | 256 × 256 | 72  
icon_256 × 256@2x.png | 512 × 512 | 144  
icon_512 × 512.png | 512 × 512 | 72  
icon_512 × 512@2x.png | 1024 × 1024 | 144  
  
从上表可以看出，要想生成 icns 文件，就要创建上表的 10 个 png 图像。通常会准备 1024 × 1024 大小的 png
图像，可以使用任意一款图像工具（如 PS）生成不同尺寸的 png 图像，不过有点麻烦。为了方便生成这 10 个 png 图像，可以利用 sips
命令。因此我们可以编写一个 buildicns.sh 脚本文件，代码如下：

    
    
    mkdir me.iconset
    sips -z 16 16     icon1024.png --out me.iconset/icon_16x16.png
    sips -z 32 32     icon1024.png --out me.iconset/icon_16x16@2x.png
    sips -z 32 32     icon1024.png --out me.iconset/icon_32x32.png
    sips -z 64 64     icon1024.png --out me.iconset/icon_32x32@2x.png
    sips -z 128 128   icon1024.png --out me.iconset/icon_128x128.png
    sips -z 256 256   icon1024.png --out me.iconset/icon_128x128@2x.png
    sips -z 256 256   icon1024.png --out me.iconset/icon_256x256.png
    sips -z 512 512   icon1024.png --out me.iconset/icon_256x256@2x.png
    sips -z 512 512   icon1024.png --out me.iconset/icon_512x512.png
    cp icon1024.png me.iconset/icon_512x512@2x.png
    iconutil --convert icns --output me.icns me.iconset
    rm -R me.iconset
    

在上面的代码中，首先会创建一个名为 me.iconset 的目录（该目录必须以 iconset 作为扩展名），然后使用 sips 命令将 1024 ×
1024 尺寸的图像压缩成相应尺寸的图像，并将这些图像放到 me.iconset 目录中，然后使用 iconutil 命令生成 me.icns
文件，最后删除 me.iconset 目录。

接下来使用 sh buildicns.sh 命令生成 me.icns 文件，然后打开 me.icns 文件，就会看到如下图所示的效果。

![](https://images.gitbook.cn/Ft032DOEQyA9mogiF04sHKw2IEa7)

其实将 me.icns 文件改成 electron.icns，一样可以修改图标，不过这里使用 --icon 命令行参数修改图标，命令如下：

    
    
    electron-packager .  me  --icon=/Users/lining/Desktop/icns/me.icns  --electron-version=3.0.2
    

在 Windows 下修改应用程序图标相对简单，只需要找一个 ico 文件，并使用下面的命令打包即可。

    
    
    electron-packager .  me  --icon=D:\MyStudio\resources\electron\images\folder.ico  --electron-version=3.0.2
    

执行上面的命令后，会生成一个 me.app 目录，很明显，me.app 目录的图标改变了。

### 22.8 操作系统平台

通常来讲，在相应平台上使用 electron-packager 命令打包 Electron 应用，会生成该平台的包目录和相关的文件。不过使用
--platform 命令行参数可以在某个平台上生成其他平台的包目录和相关文件。

在 Mac OS X 平台下可以为 Windows 和 Linux 平台打包，不过要想生成 Windows 平台的程序包，需要使用下面的命令安装
wine，这是一个在 Mac OS X 平台下模拟 Windows 运行环境的工具。

    
    
    brew install wine
    

如果指定 --platform 命令行参数为 all，那么 electron-packager 命令就会为所有的平台打包。

    
    
    electron-packager .  me  --platform=all --electron-version=3.0.2
    

执行上面的命令后，可能需要下载相应平台的 Electron 开发包，如果成功生成 Windows、Mac OS X 和 Linux
平台的包目录，会看到如下图的目录结构。

![](https://images.gitbook.cn/FllWdnp5rgDTGMp06Srb2SgsRh3I)

其中，me-mas-x64 是 Mac OS X 平台的包目录、me-win32-x64 是 Windows 平台的包目录、me-linux-x64 是
Linux 平台的包目录，将后两个包目录复制到 Windows 和 Linux 平台上可以直接运行。

在 Windows 平台无法生成 Mac OS X 平台的包目录，但可以生成 Linux 和 Windows 平台的包目录。

\--platform 命令函数参数除了 all 外，还支持如下 4 个值。

  * darwin：Mac OS X 系统
  * Linux：Linux 系统
  * mas：与 darwin 相同，也是 Mac OS X 系统
  * win32：Windows 系统

如果不想生成所有平台的包目录，可以使用上面 4 个值，多个值之间用逗号分隔。

例如，下面的命令只生成 Windows 和 Linux 平台的：

    
    
    electron-packager .  me  --platform=win32,linux --electron-version=3.0.2
    

### 22.9 asar（打包源代码）

在默认情况下，electron-packager 会将源代码直接放到包目录中（app 目录中），不过使用 --asar 命令行参数可以将 Electron
应用中的源代码打包成 asar 文件（app.asar）。

    
    
    electron-packager .  me  --asar --platform=all  --electron-version=3.0.2
    

执行上面的命令后，会在 Resources 目录中生成一个名为 app.asar 文件，工程目录中所有的文件和子目录都被打包在这个文件中，不过 asar
文件并不保险，因为可以直接用这个命令解开。

    
    
    asar extract app.asar app
    

上面的命令将 app.asar 中的文件解压到 app 目录中，如果读者的机器上没有 asar 命令，可以使用下面的命令安装。

    
    
    npm install asar -g
    

如果想保护代码的安全，应该尽量使用混淆（就是通过工具改变源代码中的变量名、函数名，去掉回车换行等，让源代码变得不可读）或其他方式（如将敏感代码用
C++、Go 语言编写，然后在 Electron 中调用）处理。

### 22.10 嵌入元信息（Only Win 32）

在 Windows 程序中，可以将与应用程序相关的信息嵌入到 exe 文件中，如公司名称、产品名称、文件描述等，这些都需要通过
--win32metadata 的子命令行参数进行设置。下面的命令设置了一些主要的应用程序信息。

    
    
    electron-packager .  me  --asar --win32metadata.CompanyName="欧瑞科技"  --win32metadata.ProductName="我的Electron应用"  --win32metadata.FileDescription="这是一个测试程序" --win32metadata.OriginalFilename="abcd.exe"  --electron-version=3.0.2
    

命令行参数的描述如下：

  * \--win32metadata.CompanyName，公司名称
  * \--win32metadata.ProductName，产品名称
  * \--win32metadata.FileDescription，文件描述
  * \--win32metadata.OriginalFilename，原始文件名

执行上面的命令后，会生成 me.exe
文件，在右键菜单中单击“属性”菜单项，然后切换到“详细信息”页面，会看到如下图所示的信息。修改的信息，除了公司名称，都会显示在上面。

![](https://images.gitbook.cn/FkV7hSkeMea5Qxt_BI19Ztgpq75G)

### 22.11 使用 electron-packager-interactive

使用 electron-packager 工具打包需要指定多个命令行参数，比较麻烦，为了方便，可以使用 electron-packager 交互工具
electron-packager-interactive，这个程序也是一个命令行工具，执行 electron-packager-interactive
后，会在控制台一步一步提示该如何去做。

使用下面的命令安装 electron-packager-interactive。

    
    
    npm install  electron-packager-interactive -g
    

执行 electron-packager-interactive 命令，会一步一步提示应该如何做，如果要保留默认值，直接按 Enter
键即可，如果需要修改默认值，直接在控制台输入新的值即可，操作过程如下图所示。

![](https://images.gitbook.cn/FqWJTRT8o1M6EDxXcrxu8AQ6KyFA)

[点击这里下载源代码](https://github.com/geekori/electron_gitchat_src)。

### 答疑与交流

为了让订阅课程的读者更快更好地掌握课程的重要知识点，我们为每个课程配备了课程学习答疑群服务，邀请作者定期答疑，尽可能保障大家学习效果。同时帮助大家克服学习拖延问题！

请添加小助手伽利略微信 GitChatty6，并将支付截图发给她，小助手会拉你进课程学习群。

