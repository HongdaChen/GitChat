### 前言

上一篇我们完成了 JWT 的生成与解析功能，并在此基础上完成了两种不同的登录接口。服务端登录成功后将 JWT
发送给客户端，供其后续访问使用。那么发送给客户端的过程中有没有风险呢？客户端收到 JWT 后该如何存储它呢？本篇带你解决这些问题，使 JWT
在客户端和服务端之间安全高效地发挥作用。

### 知识准备

#### **HTML5 Web Storage**

HTML5 Web Storage 是 HTML5 的本地存储，客户端存储 JWT 可以使用传统的 Cookies 方式，也可以使用 Web
Storage。本篇重点介绍如何通过后者进行存储。

Web Storage 包含两种方式：

  * localStorage：本地保存，只支持 String 类型，没有时间上的限制，会在关闭页面或者浏览器后仍然存在。
  * sessionStorage：基于 Session 的本地保存，只支持 String 类型，只针对一个 Session 进行数据存储，当浏览器关闭后，数据会被删除。

云笔记项目将采用 sessionStorage 的方式存储 JWT。

#### **HTTPS**

项目使用 JWT 后，实现了无状态化，每次请求只需要携带一个 JWT 就可以证实用户身份，那么一旦 JWT 在 HTTP
传输过程中被恶意拦截，别人就可以盗用你的身份进行非法操作了，因此现在各大网站都采用了 HTTPS 的访问方式。

> HTTPS （全称：Hyper Text Transfer Protocol over SecureSocket Layer），是以安全为目标的
> HTTP 通道，在 HTTP 的基础上通过传输加密和身份认证保证了传输过程的安全性。HTTPS 在HTTP 的基础下加入 SSL 层，HTTPS
> 的安全基础是 SSL，因此加密的详细内容就需要 SSL。——来自百度百科

### 生成 HTTPS 证书及公私玥

#### **生成证书**

各大云服务提供商（阿里云、腾讯云等）都提供 HTTPS 服务，只需要有一个备案的域名就可以申请到免费的 HTTPS 证书。在学习过程中我们通过 Java
自己生成一个就可以了，进入到 Java bin 目录下，执行 `keytool -genkey` 命令即可，格式如下：

    
    
    keytool -genkey -alias kellerNotes -keyalg RSA -keysize 2048 -keystore /Users/yangkaile/Projects/kellerNotes.p12 -validity 365
    

参数说明：

  * -alias：指定证书的别名
  * -keyalg：指定证书的加密方式
  * -keysize：指定密钥长度
  * -keystore：指定证书保存的地址
  * -validity：指定证书有效期，单位为天

HTTPS 证书生成完毕后，会提示是否将其迁移到行业标准格式 PKCS12，按照提示的命令迁移一下就好了，如：

    
    
    keytool -importkeystore -srckeystore /Users/yangkaile/Projects/kellerNotes.p12 -destkeystore /Users/yangkaile/Projects/kellerNotes.p12 -deststoretype pkcs12
    

证书生成过程中需要设置密码，该密码在生成证书的公私钥时也要用到。证书生成过程中会有一些问题需要回答，自行填写就好，如图所示：

![生成Https证书](https://images.gitbook.cn/788e4d20-72aa-11ea-aa9c-4b75f9a497aa)

#### **生成公私钥**

证书生成完毕后，还需要根据证书生成公钥和私钥，以备后续使用。同样在 Java bin 目录下执行命令，格式如下：

生成公钥：

    
    
    Openssl pkcs12 -in /Users/yangkaile/Projects/kellerNotes.p12 -clcerts -out /Users/yangkaile/Projects/kellerNotes_publicKey.pem
    

生成私钥：

    
    
    Openssl pkcs12 -in /Users/yangkaile/Projects/kellerNotes.p12 -nodes -out /Users/yangkaile/Projects/kellerNotes_privateKey.pem
    

这两个命令都要输入生成证书时设置的密码，如图所示：

![生成公私钥](https://images.gitbook.cn/82b3a700-72aa-11ea-ad55-4b5f9f0c99bb)

### 服务端升级 HTTPS

#### **Spring Boot 配置 HTTPS**

在 Spring Boot 项目中配置 HTTPS 很简单，将生成的证书拷贝到 application.properties 同级目录下，然后在
application.properties 中添加如下配置即可：

    
    
    ## 更改端口为 443
    server.port=443
    ## 指定证书路径
    server.ssl.key-store=classpath:kellerNotes.p12
    ## 指定证书别名
    server.ssl.key-alias=kellerNotes
    ## 指定证书密码
    server.ssl.key-store-password=2221167890
    

这样配置完毕后，从用浏览器直接访问接口 **1001** ：

    
    
    https://localhost/base/getCodeForRegister?email=2221167890@qq.com
    

会提示不安全，这是因为我们的证书是自己生成的，没有经过浏览器的安全认证，点开”高级“按钮，选择仍然信任该网站即可，如图所示：

使用浏览器第一次访问，提示证书不安全：

![浏览器提示错误](https://images.gitbook.cn/a416dcf0-72aa-11ea-9d30-373ba420c527)

添加信任后，可正常访问接口，那么这样就是安全的访问传输方式了吗？能看出有什么区别吗？打开浏览器控制台（火狐为例：右上角菜单图标 -> Web 开发者 ->
Web 控制台 -> 网络选项卡 -> 安全性选项卡）就看出不一样的了，如下图：

![浏览器访问成功](https://images.gitbook.cn/b2b00bb0-72aa-11ea-9c78-13c8e455a47a)

在这个 GET 请求的”安全性“中可以看到该连接使用了 TLSv1.2 协议、RSA-PKCS1-SHA256
加密方式。同时能看到证书的相关信息，证书里面的内容就是我们上一步生成证书时指定的。

#### **RestTemplate 配置 HTTPS**

由于云笔记项目使用了 RestTemplate 进行 HTTP 请求的转发，因此需要对 RestTemplate 进行设置，使其支持
HTTPS，设置方式如下。

1\. 创建一个工厂类 **HttpsClientRequestFactory** 继承
`org.springframework.http.client.SimpleClientHttpRequestFactory` 类以支持 SSL
连接，代码示例：

    
    
    public class HttpsClientRequestFactory extends SimpleClientHttpRequestFactory {
          /**
           * 重写 prepareConnection 方法，对连接做预处理
           */
        @Override
        protected void prepareConnection(HttpURLConnection connection, String httpMethod) {
            try {
                  //校验要处理的连接是一个正确的 Https 连接
                if (!(connection instanceof HttpsURLConnection)) {
                    throw new RuntimeException("An instance of HttpsURLConnection is expected");
                }
                            //声明一个 Https 连接
                HttpsURLConnection httpsConnection = (HttpsURLConnection) connection;
                            //定义证书
                TrustManager[] trustAllCerts = new TrustManager[]{
                        new X509TrustManager() {
                            ... ...
                };
                //获取一个 SSLContext 实例，用于设置 SSL 连接参数
                SSLContext sslContext = SSLContext.getInstance("TLS");
                //初始化 SSLContext 实例
                sslContext.init(null, trustAllCerts, new java.security.SecureRandom());
                //设置 https 连接
                httpsConnection.setSSLSocketFactory(new MyCustomSSLSocketFactory(sslContext.getSocketFactory()));
                httpsConnection.setHostnameVerifier(
                    ... ...
                });
    
                super.prepareConnection(httpsConnection, httpMethod);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    

2\. 使用 **HttpsClientRequestFactory** 实例化 RestTemplate。

将启动类 **KellerApplication** 中 `RestTemplate restTemplate = new RestTemplate()`
改为 `RestTemplate restTemplate = new RestTemplate(new
HttpsClientRequestFactory())`。

至此，Spring Boot 项目及项目中的 RestTemplate 均改为 HTTPS 协议了。

### 前端升级 HTTPS 及实现登录功能

#### **修改项目配置**

在 vue.config.js 配置文件中，将项目代理配置 `target: 'http://127.0.0.1:8080'` 修改为 `target:
'https://127.0.0.1'` 即可，修改后的代码如下：

    
    
    proxy: {
                '/api': {
                    target: 'https://127.0.0.1',
                    changeOrigin: true,
                    secure: false
                },
                '/base': {
                    target: 'https://127.0.0.1',
                    changeOrigin: true,
                    secure: false
                }
            }
    

#### **登录界面设计**

登录页面效果设计如下：

![](https://images.gitbook.cn/f2399c10-72aa-11ea-b3fa-5b760b39398d)

根据效果图，从上到下对页面进行分析：

  * 第一部分：页面标题 **KellerNotes 登录** ，居中显示
  * 第二部分：表单，输入框、按钮
  * 第三部分：按钮或超链接，用于跳转到其他页面

在页面设计上，没有新的功能点，按照设计图一步步实现就可以了。

#### **登录功能实现**

以账号密码的登录为例，需要的数据格式如下：

    
    
    //要登录的用户    
    user: {
                      //邮箱
                        email:null,
              //密码
                        password:null,
              //邮箱格式是否验证通过，用于控制是否显示邮箱格式错误的提示
                        emailChecked:true,
              //密码格式是否验证通过，用于控制是否显示密码格式错误的提示
                        passwordChecked:true
                    }
    

点击登录按钮后，需要做的工作是：

验证邮箱和密码格式：

  * 如果验证不通过，显示相应的错误提示
  * 如果验证通过，使用 Axios 发起登录请求

解析登录请求的应答，判断业务状态是否成功：

  * 如果业务状态失败，提示错误信息
  * 否则保存 JWT，跳转到用户主页

登录操作的完整代码如下：

    
    
    //登录操作
                handleLogon(){
            //验证邮箱
                    this.user.emailChecked = format.isEmail(this.user.email);
            //验证密码
                    this.user.passwordChecked = format.isPassword(this.user.password);
                    //邮箱和密码都校验通过后发送登录请求
                    if (this.user.emailChecked && this.user.passwordChecked) {
                        req_logon(this.user).then(response =>{
                //获取到返回的结果
                            let{
                                data,
                                message,
                                success
                            } = response;
                //如果业务状态不是成功，提示错误信息
                            if(success != 0){
                                this.$message.error(message);
                            }else{
                  //业务状态成功，保存 JWT
                                window.localStorage.setItem("token",data);
                  //跳转到用户主页
                                this.$router.push({ path: '/Home' });
                            }
                        });
                    }
                }
    

至此，账号密码登录功能实现完毕。验证码登录功能和账号密码登录相似，在这里就不一一赘述了，希望大家参照已有的代码自己练习一下：

  * 参照注册功能中的 **获取注册验证码** 功能实现 **获取登录验证码** 功能。
  * 参照本篇的 **账号密码登录** 功能，实现 **验证码登录功能** 。

### Postman 升级 HTTPS

当服务端升级到 HTTPS 协议后，测试工具如 Postman 就无法正常对接口访问了，如图：

![Postman提示错误](https://images.gitbook.cn/19ae5150-72ab-11ea-9d30-373ba420c527)

访问 getCodeForRegister 接口时无法拿到应答，它提示我们要关闭 SSL 验证、配置证书。

第一步，关闭 SSL 验证，在 SETTINGS -> General 设置面板中关闭 SSL certificate verification 开关即可：

![Postman关闭SSL验证](https://images.gitbook.cn/5f135ce0-72ab-11ea-9d30-373ba420c527)

第二步，配置 HTTPS 证书的公钥和私钥，这里个公钥和私钥就是我们在生成 HTTPS 证书时生成的公私钥：

![添加客户端公私钥](https://images.gitbook.cn/68564600-72ab-11ea-b086-d1dd6d1a07b6)

然后，就可以正常访问了：

![Postman访问成功](https://images.gitbook.cn/70902390-72ab-11ea-b3fa-5b760b39398d)

### 源码地址

本篇完整的源码地址如下：

> <https://github.com/tianlanlandelan/KellerNotes/tree/master/14.登录2>

### 小结

本篇带领大家将服务端、前端代码都升级到 HTTPS 协议，以保证 JWT 在传输的安全性，同时完成前端账号密码登录功能，将服务端生成的 JWT
保存在前端，这样就形成了一个安全、完整的账号认证体系。在最后讲解了如何配置 Postman 使其支持 HTTPS
协议，以方便我们在后续的开发过程中对接口进行测试。

