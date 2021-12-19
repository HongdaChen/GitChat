### 前言

在 Web 项目中，文件上传功能几乎是必不可少的，实现的技术有很多，这节课我们来学习如何使用 Spring MVC 框架完成文件的上传以及下载。

首先我们来学习文件上传，这里介绍两种上传方式：单文件上传和多文件批量上传。

### 单文件上传

（1）底层使用的是 Apache fileupload 组件完成上传功能，Spring MVC 只是对其进行了封装，让开发者使用起来更加方便，因此首先需要在
pom.xml 中引入 fileupload 组件依赖。

    
    
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
        <version>1.3.2</version>
    </dependency>
    <dependency>
        <groupId>commons-fileupload</groupId>
        <artifactId>commons-fileupload</artifactId>
        <version>1.2.1</version>
    </dependency>
    

（2）JSP 页面做如下处理：

  * input 的 type 设置为 file
  * form 表单的 method 设置为 post（get 请求只会将文件名传给后台）
  * form 表单的 enctype 设置为 multipart/form-data，以二进制的形式传输数据

    
    
    <form action="upload" method="post" enctype="multipart/form-data">
        <input type="file" name="img">
        <input type="submit" name="提交">
    </form><br /> 
    <c:if test="${filePath!=null }">
        <h1>上传的图片</h1><br /> 
        <img width="300px" src="<%=basePath %>${filePath}"/>
    </c:if>
    

如果上传成功，返回当前页面，展示上传成功的图片，这里需要使用 JSTL 标签进行判断，在 pom.xml 中引入 JSTL 依赖。

    
    
    <dependency>
         <groupId>jstl</groupId>
         <artifactId>jstl</artifactId>
         <version>1.2</version>
    </dependency>
    <dependency>
         <groupId>taglibs</groupId>
         <artifactId>standard</artifactId>
         <version>1.1.2</version>
    </dependency>
    

（3）完成业务方法，使用 MultipartFile 对象作为参数，接收前端发送过来的文件，并完成上传操作。

    
    
    @RequestMapping(value="/upload", method = RequestMethod.POST)
    public String upload(@RequestParam(value="img")MultipartFile img, HttpServletRequest request)
      throws Exception {
      //getSize() 方法获取文件的大小来判断是否有上传文件
      if (img.getSize() > 0) {
        //获取保存上传文件的 file 文件夹绝对路径
        String path = request.getSession().getServletContext().getRealPath("file");
        //获取上传文件名
        String fileName = img.getOriginalFilename();
        File file = new File(path, fileName);
        img.transferTo(file);
        //保存上传之后的文件路径
        request.setAttribute("filePath", "file/"+fileName);
        return "upload";
      }
      return "error";
    }
    

（4）在 springmvc.xml 中配置 CommonsMultipartResolver

    
    
    <!-- 配置 CommonsMultipartResolver bean，id 必须是 multipartResolver -->
    <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <!-- 处理文件名中文乱码 -->
        <property name="defaultEncoding" value="utf-8"/>
        <!-- 设置多文件上传，总大小上限，不设置默认没有限制，单位为字节，1M=1*1024*1024 -->
        <property name="maxUploadSize" value="1048576"/>
        <!-- 设置每个上传文件的大小上限 -->
        <property name="maxUploadSizePerFile" value="1048576"/>
    </bean>
    
    <!-- 设置异常解析器 -->
    <bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">
        <property name="defaultErrorView" value="/error.jsp"/>
    </bean>
    

（5）运行，如下图所示：

![](https://images.gitbook.cn/8d0b9800-9a18-11e8-9724-fb3ff7de9e38)

上传成功如下图所示：

![](https://images.gitbook.cn/91b8cc60-9a18-11e8-bd2f-43e393597943)

### 多文件上传

（1）JSP

    
    
    <form action="uploads" method="post" enctype="multipart/form-data">
        file1:<input type="file" name="imgs"><br />
        file2:<input type="file" name="imgs"><br /> 
        file3:<input type="file" name="imgs"><br />  
        <input type="submit" name="提交">
    </form>
    <c:if test="${filePaths!=null }">
        <h1>上传的图片</h1><br /> 
        <c:forEach items="${filePaths }" var="filePath">
            <img width="300px" src="<%=basePath %>${filePath}"/>
        </c:forEach>
    </c:if>
    

（2）业务方法，使用 MultipartFile 数组对象接收上传的多个文件

    
    
    @RequestMapping(value="/uploads", method = RequestMethod.POST)
    public String uploads(@RequestParam MultipartFile[] imgs, HttpServletRequest request)
      throws Exception {
      //创建集合，保存上传后的文件路径
      List<String> filePaths = new ArrayList<String>();
      for (MultipartFile img : imgs) {
        if (img.getSize() > 0) {
          String path = request.getSession().getServletContext().getRealPath("file");
          String fileName = img.getOriginalFilename();
          File file = new File(path, fileName);
          filePaths.add("file/"+fileName);
          img.transferTo(file);
        }
      }
      request.setAttribute("filePaths", filePaths);
      return "uploads";
    }
    

（3）运行，如下图所示

![](https://images.gitbook.cn/a8614640-9a18-11e8-9724-fb3ff7de9e38)

上传成功如下图所示

![](https://images.gitbook.cn/ac7e6fa0-9a18-11e8-9724-fb3ff7de9e38)

完成了文件上传，接下来我们学习文件下载的具体使用。

（1）在 JSP 页面中使用超链接，下载之前上传的 logo.png。

    
    
    <a href="download?fileName=logo.png">下载图片</a>
    

（2）业务方法。

    
    
    @RequestMapping("/download")
    public void downloadFile(String fileName,HttpServletRequest request,
                             HttpServletResponse response){
      if(fileName!=null){
        //获取 file 绝对路径
        String realPath = request.getServletContext().getRealPath("file/");
        File file = new File(realPath,fileName);
        OutputStream out = null;
        if(file.exists()){
          //设置下载完毕不打开文件 
          response.setContentType("application/force-download");
          //设置文件名 
          response.setHeader("Content-Disposition", "attachment;filename="+fileName);
          try {
            out = response.getOutputStream();
            out.write(FileUtils.readFileToByteArray(file));
            out.flush();
          } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
          }finally{
            if(out != null){
              try {
                out.close();
              } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
              }
            }
          }                    
        }           
      }           
    }
    

（3）运行，如下所示。

![](https://images.gitbook.cn/b6a92b40-9a19-11e8-8334-9bfa28241acd)

下载成功如下所示。

![](https://images.gitbook.cn/bb561180-9a19-11e8-992f-9dfb28d2b53f)

### 总结

本节课我们讲解了 Spring MVC 框架对于文件上传和下载的支持，文件上传和下载底层是通过 IO 流完成的，上传就是将客户端的资源通过 IO
流写入服务端，下载刚好相反，将服务端资源通过 IO 流写入客户端。Spring MVC 提供了一套完善的上传下载机制，可以有效地简化开发步骤。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

[请单击这里下载源码](https://github.com/southwind9801/Spring-MVC-UploadDownload.git)

