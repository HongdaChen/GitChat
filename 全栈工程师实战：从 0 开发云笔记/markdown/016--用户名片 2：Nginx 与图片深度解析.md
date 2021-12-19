### 前言

上一篇，我们实现了图片的上传及压缩功能，将上传好的图片存在在了指定目录中。本篇将带领大家搭建一个 Nginx
文件服务器，用来访问上传的文件，同时会补充一些图片信息解析的知识。

### 知识准备

本篇内容主要涉及到 Nginx 搭建文件服务器、图片信息的解析，因此，在学习之前，我们先了解一下相关的知识。

#### **Nginx**

Nginx 官网的介绍如下：

> NGINX is an open source web server used by more than 400 million websites
> and over 66% of the world’s top 10,000 websites. NGINX Plus, built on top of
> open source NGINX, adds enterprise‑grade capabilities such as load balancing
> with application‑aware health checks, content caching, security controls,
> and rich application monitoring and management capabilities.

这段话是说：Nginx 是一个开放源代码的 Web 服务器，有超过 4 亿个网站使用，占世界前 10000 个网站的 66%。Nginx Plus 是在开源
Nginx 的基础上构建的，它增加了企业级功能，如通过应用程序感知的运行状况检查实现负载平衡、内容缓存、安全控制以及丰富的应用程序监视和管理功能。

Nginx 官网地址：

> <https://www.nginx.com>

![Nginx](https://images.gitbook.cn/42e139f0-7951-11ea-b471-817fff5e2efb)

Nginx 服务器稳定性好，功能强大，主要支持：

  * 正向代理：
    * 用户向 Nginx 服务器发出请求，并指定目标服务器
    * Nginx 将用户请求转交给目标服务器
    * Nginx 将目标服务器返回的内容返回给客户端
  * Rewriter：请求地址重定向
  * 热部署：在不打断用户请求的情况下更新版本
  * 静态文件处理：作为文件服务器使用，用户可以从 Nginx 上获取到图片、视频等文件资源
  * 反向代理：
    * 用户发送请求到 Nginx
    * Nginx 将请求转发到实际处理的服务器
    * Nginx 将服务器返回的内容发送给客户端
  * 负载均衡：将用户请求分流，减少服务端压力

本项目中使用到了 Nginx 的静态文件处理功能。

#### **图片元数据**

图片元数据是在图片拍摄时生成的一些记录性信息，常见的是 EXIF 信息，百度百科对其做如下定义：

> EXIF 信息，是可交换图像文件的缩写，是专门为数码相机的照片设定的，可以记录数码照片的属性信息和拍摄数据。EXIF 可以附加于
> JPEG、TIFF、RIFF 等文件之中，为其增加有关数码相机拍摄信息的内容和索引图或图像处理软件的版本信息。

我们可以从图片 EXIF 信息中获取到如下信息：

  * Image Description：图像描述、来源
  * Artist：作者 
  * Make：生产者 
  * Model：设备型号
  * Date Time：日期和时间
  * Exif Offset Exif：信息位置
  * Metering Mode：测光方式、平均式测光、中央重点测光、点测光等
  * Flash：是否使用闪光灯
  * Maker Note：作者标记、说明、记录
  * Color Space：色域、色彩空间
  * ExifImage Width：图像宽度
  * ExifImage Length：图像高度
  * ……

#### **metadata-extractor**

metadata-extractor 提供了简单的 API 用于访问图片和视频中的元数据，支持大多数格式的图片和视频：

  * JPEG/PNG/GIF/BMP……
  * MP4/3GP/M4V/MOV……

官网地址：

> https://drewnoakes.com/code/exif/[](https://drewnoakes.com/code/exif/)

![metadata](https://images.gitbook.cn/7f4c7350-7951-11ea-b471-817fff5e2efb)

云笔记项目使用 Maven 方式安装依赖包：

    
    
    <dependency>
        <groupId>com.drewnoakes</groupId>
        <artifactId>metadata-extractor</artifactId>
        <version>2.11.0</version>
    </dependency>
    

### Nginx 文件服务器

Nginx 在不同的平台上有不同的安装方式：

  * Windows 平台下，下载压缩后后解压，运行 EXE 文件即可
  * macOS 平台下，使用 brew 命令安装
  * Linux 平台下，使用 yum 命令安装

下面主要介绍一下大家常用的 Windows 平台和 macOS 平台的安装方式，Linux 平台参照 macOS 平台即可。

#### Windows 安装 Nginx

Windows 下安装 Nginx 只需要下载安装包解压后即可使用，具体步骤如下：

进入 Nginx 下载页面，选择 Stable version（稳定版）下的 nginx/Windows-1.X.X 版本，点击下载。

Nginx 下载地址：

> <http://nginx.org/en/download.html>

![Nginx下载](https://images.gitbook.cn/9c317420-7951-11ea-9092-0d4306e530bd)

下载完成后解压，得到如下文件夹：

直接运行 nginx.exe 文件即可完成 Nginx 的启动，Nginx 的配置文件在 conf 目录下，稍后讲解如何修改配置文件，使其成为文件服务器。

#### **macOS 安装 Nginx**

macOS 直接使用 brew 命令安装即可：

    
    
    brew install nginx
    

安装完毕后，可以使用 `nginx -v` 命令查看 Nginx 版本，使用 `nginx` 命令启动。

![NginxMac](https://images.gitbook.cn/05af4340-7953-11ea-8a3f-55ca77157595)

Mac 上常用的 Nginx 命令：

  * `nginx`：启动
  * `nginx -s reload`：重启
  * `nginx -s stop`：停止

启动完成后，访问 http://localhost:80 可看到 Nginx 欢迎页：

#### **Nginx 配置**

修改 nginx.conf，就可以修改 Nginx 配置，该文件的位置如下：

  * Windows：nginx.exe 同级目录下 conf 文件夹中
  * macOS：/usr/local/etc/nginx/nginx.conf

不同平台上该文件的配置是一样的，找到 server 配置项，按照如下格式修改：

    
    
        server {
            listen   8081;
            server_name  localhost;
    
            location /keller{
               alias /Users/yangkaile/Projects/nginx/keller;
               index index.html;
            }
    
            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }
    

配置项说明如下：

  * listen：Nginx 启动端口，和上一篇服务器配置的 nginx.url 保持一致
  * server_name：Nginx 服务名
  * location /keller：要代理的地址，这样写的意思是：访问 http://location/keller 的请求都会被处理，和上一篇服务器配置的 nginx.url 保持一致
    * alias：文件夹地址，本地文件夹地址，要和上一篇服务器配置的 nginx.path 保持一致
    * index：欢迎页

配置完成后，重启 Nginx 即可。

#### **效果测试**

修改用户头像，修改完成后，调用获取用户名片的接口，根据获取到的 protraitUrl 地址，直接在浏览器访问，如果能获取到刚才上传的图片，说明 Web
服务器和 Nginx 配置正确，如图：

![上传头像](https://images.gitbook.cn/4a224db0-7953-11ea-a9bb-7bb763c3770e)

![访问图片](https://images.gitbook.cn/50402910-7953-11ea-9df1-7f4788873a0f)

### 图片深度解析

在云笔记项目中使用到了图片的上传和访问，而这些功能在照片墙、社交网站中是最常用的。往往用到这些功能的同时还需要对图片信息进行解析，以实现图片分类保存、地点标注等。接下来就向大家简单介绍一下怎么读取到图片中的其他信息。

#### **图片元数据读取**

图片的 EXIF 信息主要在照片中有大量的保存，一般的截图、压缩图等不会保留，因此最好拿手机、相机拍摄的原图进行测试。下面以 jpg/jepg
格式的照片信息读取为例，讲解 metadata-extractor 的使用。

新建工具类 MetadataUtils.java，使用 JpegMetadataReader.readMetadata(File file)
方法即可获取到照片的元数据信息：

    
    
            // 读取 jpg、jepg 格式的照片元数据，由于元数据信息庞大，只获取其中需要关注的信息
        public static Map<String,String> readJpeg(File file) {
            Metadata metadata;
            Map<String,String> result = new HashMap<>(16);
            try {
                metadata = JpegMetadataReader.readMetadata(file);
                Iterator<Directory> it = metadata.getDirectories().iterator();
                while (it.hasNext()) {
                    Directory exif = it.next();
                    Iterator<Tag> tags = exif.getTags().iterator();
                    //遍历图片 Exif 信息的，取出需要的信息放入 Map 中
                    while (tags.hasNext()) {
                        Tag tag = tags.next();
                          // 打印出每一个 tag，可以在控制台了解一下每个 tag 存放的数据
                          System.out.println(tag);
                          // 照片高度 [Exif IFD0] Image Width - 3968 pixels 只取数值部分
                        if(IMAGE_HEIGHT.equals(tag.getTagName())){
                            result.put(IMAGE_HEIGHT,tag.getDescription().substring(0,tag.getDescription().indexOf(" ")));
                        }
                          // 照片宽度，只取数值部分
                        if(IMAGE_WIDTH.equals(tag.getTagName())){
                            result.put(IMAGE_WIDTH,tag.getDescription().substring(0,tag.getDescription().indexOf(" ")));
                        }
                          // 拍摄时间
                        if(CREATE_TIME.equals(tag.getTagName())){
                            result.put(CREATE_TIME,tag.getDescription());
                        }
                          // 纬度
                        if(LATITUDE.equals(tag.getTagName())){
                            result.put(LATITUDE,tag.getDescription());
                        }
                          // 经度
                        if(LONGITUDE.equals(tag.getTagName())){
                            result.put(LONGITUDE,tag.getDescription());
                        }
                          // 设备类型
                        if(MAKE.equals(tag.getTagName())){
                            result.put(MAKE,tag.getDescription());
                        }
                          //  设备型号
                        if(MODEL.equals(tag.getTagName())){
                            result.put(MODEL,tag.getDescription());
                        }
                    }
                }
            } catch (JpegProcessingException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
            return result;
        }
    

#### **效果测试**

新建测试方法，指定一张本地的照片：

    
    
    @Test
    public void readJpeg(){
        File jpegFile = new File("/Users/yangkaile/Pictures/IMG_20190127_163451.JPG");
            Map<String,String> map = MetadataUtils.readJpeg(jpegFile);
    
            Console.println("readJpeg");
            for(String key:map.keySet()){
                System.out.println(key+ ":" + map.get(key));
            }
    }
    

运行后得到如下结果：

![readJpeg](https://images.gitbook.cn/71df5690-7953-11ea-a9bb-7bb763c3770e)

`=======readJpeg=======` 上面是照片中所有的元数据信息，林林总总有几十条，它的下面是我们挑出来需要的信息：

  * 纬度：39° 59' 20.71"，需要在拍摄时手机打开位置信息
  * 设备型号：MHA-AL00，配合设备名能知道是使用华为Mate 9 拍摄的
  * 经度：116° 16' 20.36"，需要在拍摄时手机打开位置信息
  * 设备名：HUAWEI，华为
  * 图片高度：2976，单位是像素
  * 图片宽度：3968，单位是像素
  * 拍摄时间：2019:01:27 16:34:52

有了这些，我们就可以按照拍摄时间、大小、设备型号等进行分类或排序了，当然，如果想按照拍摄地点分类整理的话，我们还需要将经纬度转换成具体的地址。

### 位置信息获取

把经纬度转换成具体的地址，被称为逆地理编码，百度、腾讯、阿里云都提供的有免费的接口可以实现。下面介绍一下如何用阿里云的接口实现。

#### **阿里云 API**

打开阿里云市场，搜索“逆地理编码”：

> <https://market.aliyun.com>

![阿里云市场](https://images.gitbook.cn/91060a00-7953-11ea-9df1-7f4788873a0f)

打开逆地理编码，里面有接口说明：

![接口](https://images.gitbook.cn/9e7bee70-7953-11ea-a1b9-972133577f73)

按照接口说明，我们只需要完成注册、获取令牌，就可以使用了。

我们可以按照页面上的调试工具进行调试，获取到的数据格式如下：

    
    
    {
        "infocode":"10000",
        "regeocode":{
            "formatted_address":"北京市海淀区青龙桥街道颐和园",
            "addressComponent":{
                "businessAreas":[[]],
                "country":"中国",
                "province":"北京市",
                "citycode":"010",
                "city":[],
                "adcode":"110108",
                "streetNumber":{
                    "number":"甲1号",
                    "distance":"675.586",
                    "street":"六郎庄路",
                    "location":"116.278221,39.9850281",
                    "direction":"东南"},
                    "towncode":"110108013000",
                    "district":"海淀区",
                    "neighborhood":{
                        "name":"颐和园",
                        "type":"风景名胜;风景名胜;世界遗产"
                    },
                    "township":"青龙桥街道",
                    "building":{
                        "name":[],
                        "type":[]
                    }
            }
        },
        "status":"1",
        "info":"OK"
    }
    

#### **经纬度格式转换**

由于从照片中获取到的经纬度是这样格式的：`40° 5' 5.49"`，而 API 中要求上传的是浮点数，我们就需要对其进行格式转换：

    
    
    /**
     * 经纬度格式转换，从时分秒转换成 float 格式，保留小数点后 6 位
     * @param point 坐标点 格式：40° 5' 5.49"
     * @return 转换后的格式：40.084858
     */
    private static float pointToFloat (String point ) {
          //读取度数
        Double du = Double.parseDouble(point.substring(0, point.indexOf("°")).trim());
          //读取分
        Double fen = Double.parseDouble(point.substring(point.indexOf("°")+1, point.indexOf("'")).trim());
          //读取秒
        Double miao = Double.parseDouble(point.substring(point.indexOf("'")+1, point.indexOf("\"")).trim());
          //按照 60 进制换算为浮点数
        Double duStr = du + fen / 60 + miao / 60 / 60 ;
        String str = duStr.toString();
        if(str.substring(str.indexOf(".") + 1).length() > 6){
            return Float.parseFloat(str.substring(0,str.indexOf(".") + 6 + 1));
        }
        return Float.parseFloat(str);
    }
    

#### **读取地址**

使用免费申请到的 appCode 即可调用 API：

-构建一个 GET 请求

  * 请求参数中添加 appCode 和要查询的经纬度
  * 发送请求
  * 按照 JSON 格式解析出需要的地址信息

    
    
        /**
         * 阿里云地图 API 身份认证信息
         */
        private static final String APP_CODE = "c8b7ef0a794a4e7e96e6a309d150dd8c";
        /**
         * 阿里云地图 API 访问地址
         */
        private static final String URL = "https://regeo.market.alicloudapi.com/v3/geocode/regeo";
        /**
         * 根据经纬度获取地址
         * @param longitude 经度，小数点后不超过 6 位，如 116.36486
         * @param latitude  纬度，小数点后不超过 6 位，如 40.084858
         * @return 地址，如：北京市海淀区青龙桥街道颐和园
         */
        private static String getAddress(float longitude, float latitude) {
            String url = URL + "?location=" + longitude + "," + latitude;
            String result = null;
            try {
                //Header 中放入 Authorization 信息：AppCode
                HttpHeaders httpHeaders = new HttpHeaders();
                httpHeaders.add("Authorization", "APPCODE " + APP_CODE);
                RestTemplate restTemplate = RestTemplateUtils.getTemplate();
                //发送 Get 请求
                HttpEntity<String> requestEntity = new HttpEntity(null, httpHeaders);
                ResponseEntity<String> response = restTemplate.exchange(url, HttpMethod.GET, requestEntity, String.class);
                Console.println("response",response);
                if(response.getStatusCode() == HttpStatus.OK){
                    result = getAddrByBody(response.getBody());
                }
    
            } catch (Exception e) {
                e.printStackTrace();
            }
            return result;
        }
    

至此，就完成了地址的解析，在需要的时候，就可以解析照片的拍摄地点了。

#### **百度 Web 服务 API**

阿里云地图 API 使用的是高德地图数据，如果想用百度地图，可以调用百度逆地理编码接口，地址如下：

> <http://lbsyun.baidu.com/index.php?title=webapi>

![百度地图](https://images.gitbook.cn/d3a67750-7953-11ea-8a3f-55ca77157595)

接口调用方式和阿里云地图 API 一样，感兴趣的同学可以自己实现一下，同时，你会发现里面有很多好玩的 API，拿来用在自己的项目中简直不要太爽哦。

### 源码地址

本篇完整的源码地址如下：

> <https://github.com/tianlanlandelan/KellerNotes/tree/master/16.用户名片2/server>

### 小结

本篇带领大家实现了通过 Nginx 文件服务器访问图片的功能，这样做极大减轻了系统的访问压力和存储压力。在实际使用中，我们可以在服务器上分别部署 Web
服务器、Nginx 文件服务器，也可以使用七千牛、腾讯云、阿里云等提供的文件存储功能，不建议大家使用 Web 服务器承担静态资源文件的访问工作。

同时，本篇对于图片的处理做了一些延伸的知识：读取图片 EXIF 信息，并使用地图 API
根据照片的元数据获取拍摄地点。这些延伸知识可用于搭建照片墙、图片管理、社交网站等应用，希望大家在课后实际体验一下。

