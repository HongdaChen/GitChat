本文我们将集成 Redis，实现 API 鉴权机制。

### Redis 的集成

Spring Boot 集成 Redis 相当简单，只需要在 pom 里加入如下依赖即可：

    
    
    <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-data-redis</artifactId>
            </dependency>
    

由于每个模块都可能用到 Redis，因此我们可以考虑将 Redis 的依赖放到 common 工程下：

![enter image description
here](http://images.gitbook.cn/cf2da190-786e-11e8-b30f-e135213f0339)

然后创建一个类实现基本的 Redis 操作：

    
    
    @Component
    public class Redis {
    
        @Autowired
        private StringRedisTemplate template;
    
        /**
         * expire为过期时间，秒为单位
         *
         * @param key
         * @param value
         * @param expire
         */
        public void set(String key, String value, long expire) {
            template.opsForValue().set(key, value, expire, TimeUnit.SECONDS);
        }
    
        public void set(String key, String value) {
            template.opsForValue().set(key, value);
        }
    
        public Object get(String key) {
            return template.opsForValue().get(key);
        }
    
        public void delete(String key) {
            template.delete(key);
        }
    }
    

如果具体的模块需要操作 Redis 还需要在配置文件配置 Redis 的连接信息，这里我们在 Git 仓库创建一个 yaml 文件
redis.yaml，并加入以下内容：

    
    
    spring:
      redis:
        host: localhost
        port: 6379
        password:
    

最后在需要操作 Redis 的工程的 bootstrap.yml 文件中加上 Redis 配置文件名即可，如下：

    
    
    spring:
      cloud:
        config:
          #这里加入 redis
          name: user,feign,database,redis,key
          label: master
          discovery:
            enabled: true
            serviceId: config
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8888/eureka/
    

这样在工程想操作 Redis 的地方注入 Redis 类：

    
    
    @Autowired
        private Redis redis;
    

但这样启动工程会报错，原因是 CommonScan 默认从工程根目录开始扫描，我们工程的根包名是：com.lynn.xxx（其中 xxx 为工程名)，而
Redis 类在 com.lynn.common 下，因此我们需要手动指定开始扫描的包名，我们发现二者都有 com.lynn，所以指定为 comm.lynn
即可。

在每个工程的 Application 类加入如下注解：

    
    
    @SpringCloudApplication
    @ComponentScan(basePackages = "com.lynn")
    @EnableHystrixDashboard
    @EnableFeignClients
    public class Application {
    
        public static void main(String[] args) {
            SpringApplication.run(Application.class,args);
        }
    }
    

### API 鉴权

互联网发展至今，已由传统的前后端统一架构演变为如今的前后端分离架构，最初的前端网页大多由 JSP、ASP、PHP
等动态网页技术生成，前后端十分耦合，也不利于扩展。现在的前端分支很多，如 Web 前端、Android 端、iOS
端，甚至还有物联网等。前后端分离的好处就是后端只需要实现一套界面，所有前端即可通用。

前后端的传输通过 HTTP
进行传输，也带来了一些安全问题，如果抓包、模拟请求、洪水攻击、参数劫持、网络爬虫等。如何对非法请求进行有效拦截，保护合法请求的权益是这篇文章需要讨论的。

我依据多年互联网后端开发经验，总结出了以下提升网络安全的方式：

  * 采用 HTTPS 协议；
  * 密钥存储到服务端而非客户端，客户端应从服务端动态获取密钥；
  * 请求隐私接口，利用 Token 机制校验其合法性；
  * 对请求参数进行合法性校验；
  * 对请求参数进行签名认证，防止参数被篡改；
  * 对输入输出参数进行加密，客户端加密输入参数，服务端加密输出参数。

接下来，将对以上方式展开做详细说明。

#### HTTP VS HTTPS

普通的 HTTP 协议是以明文形式进行传输，不提供任何方式的数据加密，很容易解读传输报文。而 HTTPS 协议在 HTTP 基础上加入了 SSL 层，而
SSL 层通过证书来验证服务器的身份，并为浏览器和服务器之间的通信加密，保护了传输过程中的数据安全。

#### 动态密钥的获取

对于可逆加密算法，是需要通过密钥进行加解密，如果直接放到客户端，那么很容易反编译后拿到密钥，这是相当不安全的做法，因此考虑将密钥放到服务端，由服务端提供接口，让客户端动态获取密钥，具体做法如下：

  * 客户端先通过 RSA 算法生成一套客户端的公私钥对（clientPublicKey 和 clientPrivateKey）；
  * 调用 getRSA 接口，服务端会返回 serverPublicKey；
  * 客户端拿到 serverPublicKey 后，用 serverPublicKey 作为公钥，clientPublicKey 作为明文对 clientPublicKey 进行 RSA 加密，调用 getKey 接口，将加密后的 clientPublicKey 传给服务端，服务端接收到请求后会传给客户端 RSA 加密后的密钥；
  * 客户端拿到后以 clientPrivateKey 为私钥对其解密，得到最终的密钥，此流程结束。

**注：**
上述提到数据均不能保存到文件里，必须保存到内存中，因为只有保存到内存中，黑客才拿不到这些核心数据，所以每次使用获取的密钥前先判断内存中的密钥是否存在，不存在，则需要获取。

为了便于理解，我画了一个简单的流程图：

![](http://images.gitbook.cn/6f19d910-7911-11e8-b06c-81deae508e47)

那么具体是如何实现的呢，请看下面的代码（同样地，我们将这些公用方法放到 common 类库下)。

##### **全局密钥配置，故加密算法统一密钥**

    
    
    api:
      encrypt:
        key: d7b85c6e414dbcda
    

此配置的公司钥信息为测试数据，不能直接使用，请自行重新生成公私钥。

    
    
    rsa:
      publicKey: MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCcZlkHaSN0fw3CWGgzcuPeOKPdNKHdc2nR6KLXazhhzFhe78NqMrhsyNTf3651acS2lADK3CzASzH4T0bT+GnJ77joDOP+0SqubHKwAIv850lT0QxS+deuUHg2+uHYhdhIw5NCmZ0SkNalw8igP1yS+2TEIYan3lakPBvZISqRswIDAQAB
      privateKey: MIICeAIBADANBgkqhkiG9w0BAQeFAcSCAmIwggJeAgEAAoGBAJxmWQdpI3R/DcJYaDNy4944o900od1zadHootdrOGHMWF7vw2oyuGzI1N/frmxoVLaUAMrcLMBLMfhPRtP4acnvuOgM4/7RKq5scrAAi/znSVPRDFL5165QeDb64diF2EjDk0KZnRKQ1qXDyKA/XJL7ZMQhhqfeVqQ8G9khKpGzAgMBAAECgYEAj+5AkGlZj6Q9bVUez/ozahaF9tSxAbNs9xg4hDbQNHByAyxzkhALWVGZVk3rnyiEjWG3OPlW1cBdxD5w2DIMZ6oeyNPA4nehYrf42duk6AI//vd3GsdJa6Dtf2has1R+0uFrq9MRhfRunAf0w6Z9zNbiPNSd9VzKjjSvcX7OTsECQQD20kekMToC6LZaZPr1p05TLUTzXHvTcCllSeXWLsjVyn0AAME17FJRcL9VXQuSUK7PQ5Lf5+OpjrCRYsIvuZg9AkEAojdC6k3SqGnbtftLfGHMDn1fe0nTJmL05emwXgJvwToUBdytvgbTtqs0MsnuaOxMIMrBtpbhS6JiB5Idb7GArwJAfKTkmP5jFWT/8dZdBgFfhJGv6FakEjrqLMSM1QT7VzvStFWtPNYDHC2b8jfyyAkGvpSZb4ljZxUwBbuh5QgM4QJBAJDrV7+lOP62W9APqdd8M2X6gbPON3JC09EW3jaObLKupTa7eQicZsX5249IMdLQ0A43tanez3XXo0ZqNhwT8wcCQQDUubpNLwgAwN2X7kW1btQtvZW47o9CbCv+zFKJYms5WLrVpotjkrCgPeuloDAjxeHNARX8ZTVDxls6KrjLH3lT
    
    
    
     <dependency>
                <groupId>commons-codec</groupId>
                <artifactId>commons-codec</artifactId>
            </dependency>
            <dependency>
                <groupId>commons-io</groupId>
                <artifactId>commons-io</artifactId>
                <version>2.6</version>
            </dependency>
    
    
    
    public class AesEncryptUtils {
    
        private static final String KEY = "d7585fde114abcda";    
        private static final String ALGORITHMSTR = "AES/CBC/NoPadding";    public static String base64Encode(byte[] bytes) {        return Base64.encodeBase64String(bytes);
        }    public static byte[] base64Decode(String base64Code) throws Exception {        return Base64.decodeBase64(base64Code);
        }    public static byte[] aesEncryptToBytes(String content, String encryptKey) throws Exception {
            KeyGenerator kgen = KeyGenerator.getInstance("AES");
            kgen.init(128);
            Cipher cipher = Cipher.getInstance(ALGORITHMSTR);
            cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(encryptKey.getBytes(), "AES"));        return cipher.doFinal(content.getBytes("utf-8"));
        }    public static String aesEncrypt(String content, String encryptKey) throws Exception {        return base64Encode(aesEncryptToBytes(content, encryptKey));
        }    public static String aesDecryptByBytes(byte[] encryptBytes, String decryptKey) throws Exception {
            KeyGenerator kgen = KeyGenerator.getInstance("AES");
            kgen.init(128);
            Cipher cipher = Cipher.getInstance(ALGORITHMSTR);
            cipher.init(Cipher.DECRYPT_MODE, new SecretKeySpec(decryptKey.getBytes(), "AES"));        byte[] decryptBytes = cipher.doFinal(encryptBytes);        return new String(decryptBytes);
        }    public static String aesDecrypt(String encryptStr, String decryptKey) throws Exception {        return aesDecryptByBytes(base64Decode(encryptStr), decryptKey);
        }    public static void main(String[] args) throws Exception {
            String content = "{name:\"lynn\",id:1}";
            System.out.println("加密前：" + content);
    
            String encrypt = aesEncrypt(content, KEY);
            System.out.println(encrypt.length() + ":加密后：" + encrypt);
    
            String decrypt = aesDecrypt("H9pGuDMV+iJoS8YSfJ2Vx0NYN7v7YR0tMm1ze5zp0WvNEFXQPM7K0k3IDUbYr5ZIckTkTHcIX5Va/cstIPrYEK3KjfCwtOG19l82u+x6soa9FzAtdL4EW5HAFMmpVJVyG3wz/XUysIRCwvoJ20ruEwk07RB3ojc1Vtns8t4kKZE=", "d7b85f6e214abcda");
            System.out.println("解密后：" + decrypt);
        }
    }public class RSAUtils {
    
        public static final String CHARSET = "UTF-8";    public static final String RSA_ALGORITHM = "RSA";    public static Map<String, String> createKeys(int keySize){        //为RSA算法创建一个KeyPairGenerator对象
            KeyPairGenerator kpg;        try{
                kpg = KeyPairGenerator.getInstance(RSA_ALGORITHM);
            }catch(NoSuchAlgorithmException e){            throw new IllegalArgumentException("No such algorithm-->[" + RSA_ALGORITHM + "]");
            }        //初始化KeyPairGenerator对象,密钥长度
            kpg.initialize(keySize);        //生成密匙对
            KeyPair keyPair = kpg.generateKeyPair();        //得到公钥
            Key publicKey = keyPair.getPublic();
            String publicKeyStr = Base64.encodeBase64String(publicKey.getEncoded());        //得到私钥
            Key privateKey = keyPair.getPrivate();
            String privateKeyStr = Base64.encodeBase64String(privateKey.getEncoded());
            Map<String, String> keyPairMap = new HashMap<>(2);
            keyPairMap.put("publicKey", publicKeyStr);
            keyPairMap.put("privateKey", privateKeyStr);        return keyPairMap;
        }    /**
         * 得到公钥
         * @param publicKey 密钥字符串（经过base64编码）
         * @throws Exception
         */
        public static RSAPublicKey getPublicKey(String publicKey) throws NoSuchAlgorithmException, InvalidKeySpecException {        //通过X509编码的Key指令获得公钥对象
            KeyFactory keyFactory = KeyFactory.getInstance(RSA_ALGORITHM);
            X509EncodedKeySpec x509KeySpec = new X509EncodedKeySpec(Base64.decodeBase64(publicKey));
            RSAPublicKey key = (RSAPublicKey) keyFactory.generatePublic(x509KeySpec);        return key;
        }    /**
         * 得到私钥
         * @param privateKey 密钥字符串（经过base64编码）
         * @throws Exception
         */
        public static RSAPrivateKey getPrivateKey(String privateKey) throws NoSuchAlgorithmException, InvalidKeySpecException {        //通过PKCS#8编码的Key指令获得私钥对象
            KeyFactory keyFactory = KeyFactory.getInstance(RSA_ALGORITHM);
            PKCS8EncodedKeySpec pkcs8KeySpec = new PKCS8EncodedKeySpec(Base64.decodeBase64(privateKey));
            RSAPrivateKey key = (RSAPrivateKey) keyFactory.generatePrivate(pkcs8KeySpec);        return key;
        }    /**
         * 公钥加密
         * @param data
         * @param publicKey
         * @return
         */
        public static String publicEncrypt(String data, RSAPublicKey publicKey){        try{
                Cipher cipher = Cipher.getInstance(RSA_ALGORITHM);
                cipher.init(Cipher.ENCRYPT_MODE, publicKey);            return Base64.encodeBase64String(rsaSplitCodec(cipher, Cipher.ENCRYPT_MODE, data.getBytes(CHARSET), publicKey.getModulus().bitLength()));
            }catch(Exception e){            throw new RuntimeException("加密字符串[" + data + "]时遇到异常", e);
            }
        }    /**
         * 私钥解密
         * @param data
         * @param privateKey
         * @return
         */
    
        public static String privateDecrypt(String data, RSAPrivateKey privateKey){        try{
                Cipher cipher = Cipher.getInstance(RSA_ALGORITHM);
                cipher.init(Cipher.DECRYPT_MODE, privateKey);            return new String(rsaSplitCodec(cipher, Cipher.DECRYPT_MODE, Base64.decodeBase64(data), privateKey.getModulus().bitLength()), CHARSET);
            }catch(Exception e){            throw new RuntimeException("解密字符串[" + data + "]时遇到异常", e);
            }
        }    /**
         * 私钥加密
         * @param data
         * @param privateKey
         * @return
         */
    
        public static String privateEncrypt(String data, RSAPrivateKey privateKey){        try{
                Cipher cipher = Cipher.getInstance(RSA_ALGORITHM);
                cipher.init(Cipher.ENCRYPT_MODE, privateKey);            return Base64.encodeBase64String(rsaSplitCodec(cipher, Cipher.ENCRYPT_MODE, data.getBytes(CHARSET), privateKey.getModulus().bitLength()));
            }catch(Exception e){            throw new RuntimeException("加密字符串[" + data + "]时遇到异常", e);
            }
        }    /**
         * 公钥解密
         * @param data
         * @param publicKey
         * @return
         */
    
        public static String publicDecrypt(String data, RSAPublicKey publicKey){        try{
                Cipher cipher = Cipher.getInstance(RSA_ALGORITHM);
                cipher.init(Cipher.DECRYPT_MODE, publicKey);            return new String(rsaSplitCodec(cipher, Cipher.DECRYPT_MODE, Base64.decodeBase64(data), publicKey.getModulus().bitLength()), CHARSET);
            }catch(Exception e){            throw new RuntimeException("解密字符串[" + data + "]时遇到异常", e);
            }
        }    private static byte[] rsaSplitCodec(Cipher cipher, int opmode, byte[] datas, int keySize){        int maxBlock = 0;        if(opmode == Cipher.DECRYPT_MODE){
                maxBlock = keySize / 8;
            }else{
                maxBlock = keySize / 8 - 11;
            }
            ByteArrayOutputStream out = new ByteArrayOutputStream();        int offSet = 0;        byte[] buff;        int i = 0;        try{            while(datas.length > offSet){                if(datas.length-offSet > maxBlock){
                        buff = cipher.doFinal(datas, offSet, maxBlock);
                    }else{
                        buff = cipher.doFinal(datas, offSet, datas.length-offSet);
                    }
                    out.write(buff, 0, buff.length);
                    i++;
                    offSet = i * maxBlock;
                }
            }catch(Exception e){            throw new RuntimeException("加解密阀值为["+maxBlock+"]的数据时发生异常", e);
            }        byte[] resultDatas = out.toByteArray();
            IOUtils.closeQuietly(out);        return resultDatas;
        }    public static void main(String[] args) throws Exception{
            Map<String, String> keyMap = RSAUtils.createKeys(1024);
            String  publicKey = keyMap.get("publicKey");
            String  privateKey = "MIICeAIBADANBgkqhkiG9w0BAQEFAASCAmIwggJeAgEAAoGBAJxmWQdpI3R/DcJYaDNy4944o900od1zadHootdrOGHMWF7vw2oyuGzI1N/frmxoVLaUAMrcLMBLMfhPRtP4acnvuOgM4/7RKq5scrAAi/znSVPRDFL5165QeDb64diF2EjDk0KZnRKQ1qXDyKA/XJL7ZMQhhqfeVqQ8G9khKpGzAgMBAAECgYEAj+5AkGlZj6Q9bVUez/ozahaF9tSxAbNs9xg4hDbQNHByAyxzkhALWVGZVk3rnyiEjWG3OPlW1cBdxD5w2DIMZ6oeyNPA4nehYrf42duk6AI//vd3GsdJa6Dtf2has1R+0uFrq9MRhfRunAf0w6Z9zNbiPNSd9VzKjjSvcX7OTsECQQD20kekMToC6LZaZPr1p05TLUTzXHvTcCllSeXWLsjVyn0AAME17FJRcL9VXQuSUK7PQ5Lf5+OpjrCRYsIvuZg9AkEAojdC6k3SqGnbtftLfGHMDn1fe0nTJmL05emwXgJvwToUBdytvgbTtqs0MsnuaOxMIMrBtpbhS6JiB5Idb7GArwJAfKTkmP5jFWT/8dZdBgFfhJGv6FYkEjrqLMSM1QT7VzvStFWtPNYDHC2b8jfyyAkGvpSZb4ljZxUwBbuh5QgM4QJBAJDrV7+lOP62W9APqdd8M2X6gbPON3JC09EW3jaObLKupTa7eQicZsX5249IMdLQ0A43tanez3XXo0ZqNhwT8wcCQQDUubpNLwgAwN2X7kW1btQtvZW47o9CbCv+zFKJYms5WLrVpotjkrCgPeuloDAjxeHNARX8ZTVDxls6KrjLH3lT";
            System.out.println("公钥: \n\r" + publicKey);
            System.out.println("私钥： \n\r" + privateKey);
    
            System.out.println("公钥加密——私钥解密");
            String str = "站在大明门前守卫的禁卫军，事先没有接到\n" +                "有关的命令，但看到大批盛装的官员来临，也就\n" +                "以为确系举行大典，因而未加询问。进大明门即\n" +                "为皇城。文武百官看到端门午门之前气氛平静，\n" +                "城楼上下也无朝会的迹象，既无几案，站队点名\n" +                "的御史和御前侍卫“大汉将军”也不见踪影，不免\n" +                "心中揣测，互相询问：所谓午朝是否讹传？";
            System.out.println("\r明文：\r\n" + str);
            System.out.println("\r明文大小：\r\n" + str.getBytes().length);
            String encodedData = RSAUtils.publicEncrypt(str, RSAUtils.getPublicKey(publicKey));
            System.out.println("密文：\r\n" + encodedData);
            String decodedData = RSAUtils.privateDecrypt("X4hHPa9NjPd5QJGPus+4+hWmOzbWg7oCJ1+Vc+7dHW81nEhkYnJpFyV5xcDkg70N2Mym+YAJ1PvYY9sQWf9/EkUE61TpUKBmDaGWLjEr3A1f9cKIelqLKLsJGdXEOr7Z55k4vYFvA7N3Vf5KQo3NrouvIT4wR+SjH4tDQ8tNh3JH8BvXLtXqGa2TCK2z1AzHNgYzcLCrqDasd7UDHRPZPiW4thktM/whjBn0tU9B/kKjAjLuYttKLEmy5nT7v7u16aZ6ehkk+kzvuCXF%2B3RsqraISDPbsTki2agJyqsycRx3w7CvKRyUbZhFaNcWigOwmcbZVoiom+ldh7Vh6HYqDA==", RSAUtils.getPrivateKey(privateKey));
            System.out.println("解密后文字: \r\n" + decodedData);
    
        }
    }/**
     * 私钥输入参数（其实就是客户端通过服务端返回的公钥加密后的客户端自己生成的公钥）
     */public class KeyRequest {
    
        /**
         * 客户端自己生成的加密后公钥
         */
        @NotNull
        private String clientEncryptPublicKey;    public String getClientEncryptPublicKey() {        return clientEncryptPublicKey;
        }    public void setClientEncryptPublicKey(String clientEncryptPublicKey) {        this.clientEncryptPublicKey = clientEncryptPublicKey;
        }
    }/**
     * RSA生成的公私钥输出参数
     */public class RSAResponse extends BaseResponse{
    
        private String serverPublicKey;    private String serverPrivateKey;    public static class Builder{
            private String serverPublicKey;        private String serverPrivateKey;        public Builder setServerPublicKey(String serverPublicKey){            this.serverPublicKey = serverPublicKey;            return this;
            }        public Builder setServerPrivateKey(String serverPrivateKey){            this.serverPrivateKey = serverPrivateKey;            return this;
            }        public RSAResponse build(){            return new RSAResponse(this);
            }
    
        }    public static Builder options(){        return new Builder();
        }    public RSAResponse(Builder builder){        this.serverPrivateKey = builder.serverPrivateKey;        this.serverPublicKey = builder.serverPublicKey;
        }    public String getServerPrivateKey() {        return serverPrivateKey;
        }    public String getServerPublicKey() {        return serverPublicKey;
        }
    }/**
     * 私钥输出参数
     */public class KeyResponse extends BaseResponse{
    
        /**
         * 整个系统所有加密算法共用的密钥
         */
        private String key;    public static class Builder{
            private String key;        public Builder setKey(String key){            this.key = key;            return this;
            }        public KeyResponse build(){            return new KeyResponse(this);
            }
        }    public static Builder options(){        return new Builder();
        }    private KeyResponse(Builder builder){        this.key = builder.key;
        }    public String getKey() {        return key;
        }
    
    }/**
     * API传输加解密相关接口
     */public interface EncryptOpenService {
    
        /**
         * 生成RSA公私钥
         * @return
         */
        SingleResult<RSAResponse> getRSA();    /**
         * 获得加解密用的密钥
         * @param request
         * @return
         */
        SingleResult<KeyResponse> getKey(KeyRequest request) throws Exception;
    }
    @Servicepublic class EncryptOpenServiceImpl implements EncryptOpenService{
    
        @Value("${rsa.publicKey}")    private String publicKey;    @Value("${rsa.privateKey}")    private String privateKey;    @Value("${api.encrypt.key}")    private String key;    @Override
        public SingleResult<RSAResponse> getRSA() {
            RSAResponse response = RSAResponse.options()
                    .setServerPublicKey(publicKey)
                    .build();        return SingleResult.buildSuccess(response);
        }    @Override
        public SingleResult<KeyResponse> getKey(KeyRequest request)throws Exception {
            String clientPublicKey = RSAUtils.privateDecrypt(request.getClientEncryptPublicKey(), RSAUtils.getPrivateKey(privateKey));
            String encryptKey = RSAUtils.publicEncrypt(key,RSAUtils.getPublicKey(clientPublicKey));
            KeyResponse response = KeyResponse.options()
                    .setKey(encryptKey)
                    .build();        return SingleResult.buildSuccess(response);
        }
    }
    @RestController
    @RequestMapping("open/encrypt")
    public class EncryptController {
    
        @Autowired
        private EncryptOpenService encryptOpenService;    
        @RequestMapping(value = "getRSA",method = RequestMethod.POST)    //@DisabledEncrypt
        public SingleResult<RSAResponse> getRSA(){        
            return encryptOpenService.getRSA();
        }    
        @RequestMapping(value = "getKey",method = RequestMethod.POST)    //@DisabledEncrypt
        public SingleResult<KeyResponse> getKey(@Valid @RequestBody KeyRequest request)throws Exception{        
            return encryptOpenService.getKey(request);
        }
    }
    

##### **接口请求的合法性校验**

对于一些隐私接口（即必须要登录才能调用的接口），我们需要校验其合法性，即只有登录用户才能成功调用，具体思路如下：

  * 调用登录或注册接口成功后，服务端会返回 Token（设置较短有效时间）和 refreshToken（设定较长有效时间）；
  * 隐私接口每次请求接口在请求头带上 Token，如 header(“token”,token)，若服务端 返回 403 错误，则调用 refreshToken 接口获取新的 Token 重新调用接口，若 refreshToken 接口继续返回 403，则跳转到登录界面。 

这种算法较为简单，这里就不写出具体实现了。

### 输入参数的合法性校验

一般情况下，客户端会进行参数的合法性校验，这个只是为了减轻服务端的压力，针对于普通用户做的校验，如果黑客通过直接调用接口地址，就可绕过客户端的校验，这时要求我们服务端也应该做同样的校验。

SpringMVC 提供了专门用于校验的注解，我们通过 AOP 即可实现统一的参数校验，下面请看代码：

    
    
    <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-aop</artifactId>
            </dependency>
    
    
    
    @Aspect
    @Component
    public class WebExceptionAspect {
    
        private static final Logger logger = LoggerFactory.getLogger(WebExceptionAspect.class);    //凡是注解了RequestMapping的方法都被拦截
        @Pointcut("@annotation(org.springframework.web.bind.annotation.RequestMapping)")    private void webPointcut() {
        }    /**
         * 拦截 web 层异常，记录异常日志，并返回友好信息到前端 目前只拦截 Exception，是否要拦截 Error 需再做考虑
         *
         * @param e
         *            异常对象
         */
        @AfterThrowing(pointcut = "webPointcut()", throwing = "e")    public void handleThrowing(Exception e) {
            e.printStackTrace();
            logger.error("发现异常！" + e.getMessage());
            logger.error(JSON.toJSONString(e.getStackTrace()));        try {            if(StringUtils.isEmpty(e.getMessage())){
                    writeContent(JSON.toJSONString(SingleResult.buildFailure()));
                }else {
                    writeContent(JSON.toJSONString(SingleResult.buildFailure(Code.ERROR,e.getMessage())));
                }
            }catch (Exception ex){
                ex.printStackTrace();
            }
        }    /**
         * 将内容输出到浏览器
         *
         * @param content
         *            输出内容
         */
        private void writeContent(String content)throws Exception {
            HttpServletResponse response = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes())
                    .getResponse();
            response.reset();
            response.setCharacterEncoding("UTF-8");
            response.setHeader("Content-Type", "text/plain;charset=UTF-8");
            response.setHeader("icop-content-type", "exception");
            response.getWriter().print(content);
            response.getWriter().close();
        }
    }
    

在 Controller 提供共有方法：

    
    
    protected void validate(BindingResult result){        
            if(result.hasFieldErrors()){
                List<FieldError> errorList = result.getFieldErrors();
                errorList.stream().forEach(item -> Assert.isTrue(false,item.getDefaultMessage()));
            }
        }
    

每个接口的输入参数都需要加上 `@Valid` 注解，并且在参数后面加上 BindResult 类：

    
    
    @RequestMapping(value = "/hello",method = RequestMethod.POST)    
    public SingleResult<String> hello(@Valid @RequestBody TestRequest request, BindingResult result){
            validate(result);        r
            eturn "name="+name;
    
    public class TestRequest{
    
        @NotNull(message = "name不能为空")    private String name;    public String getName() {        return name;
        }    public void setName(String name) {        this.name = name;
        }
    }
    

#### 输入参数签名认证

我们请求的接口是通过 HTTP/HTTPS
传输的，一旦参数被拦截，很有可能被黑客篡改，并传回给服务端，为了防止这种情况发生，我们需要对参数进行签名认证，保证传回的参数是合法性，具体思路如下。

请求接口前，将 Token、Timstamp 和接口需要的参数按照 ASCII 升序排列，拼接成
`url=key1=value1&key2=value2`，如 `name=xxx&timestamp=xxx&token=xxx`，进行
MD5（url+salt），得到 Signature，将 Token、Signature、Timestamp 放到请求头传给服务端，如
header(“token”,token)、header(“timestamp”,timestamp),header(“signature”,signature)。

**注：** salt 即为动态获取的密钥。

下面请看具体的实现，应该在拦截器里统一处理：

    
    
    public class ApiInterceptor implements HandlerInterceptor {
    
        private static final Logger logger = LoggerFactory.getLogger(ApiInterceptor.class);    private String salt="ed4ffcd453efab32";    @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object o) throws Exception {
            logger.info("进入拦截器");
            request.setCharacterEncoding("UTF-8");
            response.setCharacterEncoding("UTF-8");
            response.setHeader("Content-Type", "application/json;charset=utf8");
            StringBuilder urlBuilder = getUrlAuthenticationApi(request);        //这里是MD5加密算法
            String sign = MD5(urlBuilder.toString() + salt);
            String signature = request.getHeader("signature");
            logger.info("加密前传入的签名" + signature);
            logger.info("后端加密后的签名" + sign);        if(sign.equals(signature)){            return true;
            }else {            //签名错误
              response.getWriter().print("签名错误");
                response.getWriter().close();            return false;
            }
        }    private StringBuilder getUrlAuthenticationApi(HttpServletRequest request) {
            Enumeration<String> paramesNames = request.getParameterNames();
            List<String> nameList = new ArrayList<>();
            nameList.add("token");
            nameList.add("timestamp");        while (paramesNames.hasMoreElements()){
                nameList.add(paramesNames.nextElement());
            }
            StringBuilder urlBuilder = new StringBuilder();
            nameList.stream().sorted().forEach(name -> {            if ("token".equals(name) || "timestamp".equals(name)){                if("token".equals(name) && null ==request.getHeader(name)){                    return;
                    }
                    urlBuilder.append('&');
                    urlBuilder.append(name).append('=').append(request.getHeader(name));
                }            else {
                    urlBuilder.append('&');
                    urlBuilder.append(name).append('=').append(request.getParameter(name));
                }
            });
            urlBuilder.deleteCharAt(0);
            logger.info("url : " + urlBuilder.toString());        return urlBuilder;
        }    @Override
        public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
    
        }    @Override
        public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
    
        }
    }
    

#### 输入输出参数加密

为了保护数据，比如说防爬虫，需要对输入输出参数进行加密，客户端加密输入参数传回服务端，服务端解密输入参数执行请求；服务端返回数据时对其加密，客户端拿到数据后解密数据，获取最终的数据。这样，即便别人知道了参数地址，也无法模拟请求数据。

至此，基础框架就已经搭建完成，下篇我们将开始实现具体的需求。

