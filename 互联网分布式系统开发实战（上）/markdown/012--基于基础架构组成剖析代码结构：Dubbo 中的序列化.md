所谓序列化（Serialization）就是将对象转化为字节数组，用于网络传输、数据持久化或其他用途，而反序列化（Deserialization）则是把从网络、磁盘等读取的字节数组还原成原始对象，以便后续业务逻辑操作。

### 序列化基本概念和工具

序列化的方式有很多，常见的有文本和二进制两大类。XML 和 JSON 是文本类序列化方式的代表，而二进制实现的方案包括 Google 的 Protocol
Buffer 和 Facebook 的 Thrift 等。对于一个序列化实现方案而言，以下几方面的需求可以帮忙我们作出合适的选择。

#### 功能

从功能上讲，序列化的关注点在于所支持的数据结构种类以及接口友好性。数据结构种类体现在对泛型和 Map/List
等复杂数据结构的的支持，有些序列化工具并不内置这些复杂数据结构。这方面的典型例子就是后面会介绍的各种跨语言的序列化工具。因为涉及到到多语言之间的数据结构的兼容性，这些工具通常会放弃一些有些语言所不支持的、兼容性较差的复杂数据结构。

接口友好性涉及是否需要定义中间语言（Intermediate Language，IL），正如 Protocol Buffer 需要.proto
文件、Thrift 需要.thrift
文件，通过中间语言实现序列化一定程度上增加了使用的复杂度。同时，在分布式系统中，各个独立的分布式服务原则上都可以具备自身的技术体系，形成异构化系统，而异构系统实现交互就需要跨语言支持。Java
自身的序列化机制无法支持多语言也是我们使用其他各种序列化技术的一个重要原因。像前面提到过的 Protocol Buffer、Thrift 以及 Apache
Avro 都是跨语言序列化技术的代表。

#### 性能

另一方面，性能可能是我们在序列化工具选择过程中最看重的一个指标。性能指标主要包括序列化之后码流大小、序列化/反序列化速度和
CPU/内存资源占用。下表中我们列举了目前主流的一些序列化技术，可以看到在序列化和反序列化时间维度上 Alibaba 的 fastjson
具有一定优势，而从空间维度上看，相较其他技术我们可以优先选择 Protocol Buffer。

| 序列化时间 | 反序列化时间 | 大小 | 压缩后大小  
---|---|---|---|---  
Java | 8654 | 43787 | 889 | 541  
hessian | 6725 | 10460 | 501 | 313  
protocol buffer | 2964 | 1745 | _239_ | _149_  
thrift | 3177 | 1949 | 349 | 197  
json-lib | 45788 | 149741 | 485 | 263  
jackson | 3052 | 4161 | 503 | 271  
fastjson | _2595_ | _1472_ | 468 | 251  
  
### Dubbo 中的序列化实现方案

作为一种公共组件，Dubbo 中关于序列化的各个类都位于 com.alibaba.dubbo.common.serialize 包下。

Serialization 接口是序列化器的接口，定义如下。其中核心的 serialize 和 deserialize
方法分别用于执行具体的序列化和反序列化操作。

    
    
    public interface Serialization {
    
        byte getContentTypeId();
    
        String getContentType();
    
        ObjectOutput serialize(URL url, OutputStream output) throws IOException;
    
        ObjectInput deserialize(URL url, InputStream input) throws IOException;
    }
    

可以看到 serialize()和 deserialize()这两个方法对应的返回值分别是 ObjectOutput 和 ObjectInput，其中
ObjectInput 扩展自 DataInput，用于读取对象；而 ObjectOutput 扩展自
DataOutput，用于写入对象，这两个类定义如下。

    
    
    public interface ObjectInput extends DataInput {
    
        Object readObject() throws IOException, ClassNotFoundException;
    
        <T> T readObject(Class<T> cls) throws IOException, ClassNotFoundException;
    
        <T> T readObject(Class<T> cls, Type type) throws IOException, ClassNotFoundException;
    }
    
    public interface ObjectOutput extends DataOutput {
    
        void writeObject(Object obj) throws IOException;
    }
    

在 Serialization 接口的实现上，Dubbo 中默认使用的序列化实现方案基于 hessian2。Hessian
是一款优秀的序列化工具。在功能上，它支持基于二级制的数据表示形式，从而能够提供跨语言支持；在性能上，无论时间复杂度还是空间复杂度也比 Java
序列化高效很多。

在 Dubbo 中，Hessian2Serialization 实现了 Serialization 接口，我们就以
Hessian2Serialization 为例介绍 Dubbo 中具体的序列化/反序列化实现方法，Hessian2Serialization 类定义如下。

    
    
    public class Hessian2Serialization implements Serialization {
    
        public static final byte ID = 2;
    
        public byte getContentTypeId() {
            return ID;
        }
    
        public String getContentType() {
            return "x-application/hessian2";
        }
    
        public ObjectOutput serialize(URL url, OutputStream out) throws IOException {
            return new Hessian2ObjectOutput(out);
        }
    
        public ObjectInput deserialize(URL url, InputStream is) throws IOException {
            return new Hessian2ObjectInput(is);
        }
    }
    

Hessian2Serialization 中的 serialize 和 deserialize 方法分别创建了 Hessian2ObjectOutput
和 Hessian2ObjectInput 类。以 Hessian2ObjectInput 为例，使用 Hessian2Input
完成具体的反序列化操作，定义如下。

    
    
    public class Hessian2ObjectInput implements ObjectInput {
                private final Hessian2Input mH2i; 
    
        public Hessian2ObjectInput(InputStream is) {
            mH2i = new Hessian2Input(is);
            mH2i.setSerializerFactory(Hessian2SerializerFactory.SERIALIZER_FACTORY);
        }
    
        public boolean readBool() throws IOException {
            return mH2i.readBoolean();
        }
    
        public byte readByte() throws IOException {
            return (byte) mH2i.readInt();
        }
    
        public short readShort() throws IOException {
            return (short) mH2i.readInt();
        }
    
        public int readInt() throws IOException {
            return mH2i.readInt();
        }
    
        public long readLong() throws IOException {
            return mH2i.readLong();
        }
    
        public float readFloat() throws IOException {
            return (float) mH2i.readDouble();
        }
    
        public double readDouble() throws IOException {
            return mH2i.readDouble();
        }
    
        public byte[] readBytes() throws IOException {
            return mH2i.readBytes();
        }
    
        public String readUTF() throws IOException {
            return mH2i.readString();
        }
    
        public Object readObject() throws IOException {
            return mH2i.readObject();
        }
    
        @SuppressWarnings("unchecked")
        public <T> T readObject(Class<T> cls) throws IOException,
                ClassNotFoundException {
            return (T) mH2i.readObject(cls);
        }
    
        public <T> T readObject(Class<T> cls, Type type) throws IOException, ClassNotFoundException {
            return readObject(cls);
        }
    }
    

Hessian2Input 是 Hessian2 的实现库 com.caucho.hessian 中的工具类，初始化时需要设置一个
SerializerFactory，所以我们还看到这里存在一个 Hessian2SerializerFactory 专门用于设置
SerializerFactory。而在 Hessian2ObjectInput 中的各种以 read 为前缀的方法实际上都是对 Hessian2Input
相应方法的封装。Hessian2ObjectOutput 与 Hessian2ObjectInput 类，也比较简单，不再展开。

在 Dubbo 中还同时支持了 Java 二进制序列化、JSON、SOAP 文本序列化多种序列化协议，其具体实现依赖于对应的序列化框架，但整体实现过程都与
Hession 序列化方式比较类似，我们可以参考源码进行分析和学习。

### 面试题分析

#### 主流的序列化工具和框架有哪些？性能上如何进行选择？

  * 考点分析

这是关于序列化工具的一个问题，也属于比较常见和经典的一种问法。该问题考查面试者对主流的一些序列化工具的掌握情况，以及对技术选型的理解。

  * 解题思路

日常开发中被广泛使用的主流序列化工具实际上并不是很多，但确实存在一些类别。有些支持跨语言，有些在实现上需要基于中间语言。有些空间复杂都比较低，有些则胜在时间复杂度上。只要能够对工具的分类原则和特性做一些归纳，然后把主流的实现方案往这些分类和特性上去套，很多问题就迎刃而解。至于性能，通过本文中提供的对比表格，了解常见工具的时间复杂度和空间复杂度的排序即可。

  * 本文内容与建议回答

只要你学习了本文，那么这道题对你而言就是送分题。关于日常开发过程中所使用的 hessian、protocol buffer、fastjson
等框架我给出了时间复杂度和空间复杂度上的量化比较。一般只要记住这些工具在这两个维度上的大致排名即可。

### 小结与预告

本文基于 Dubbo 框架介绍了 RPC 架构中的第二个核心组件，即序列化。我们给出了 Dubbo
框架对这一组件的抽象过程以及内部的核心接口，它们位于单独的 dubbo-common 工程中。下一篇我们将关注另一个与网络通信相关的重要的话题，即
Dubbo 中的传输协议。

