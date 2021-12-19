本文开始，我们将实现具体的业务，由于篇幅问题，本文将贴出部分实例代码，其余会提供一般思路。

### 公共模块

我们的接口会分别放在不同的工程下，其中会有公共代码，在此我们考虑将公共代码抽象出来放到公共模块 common 下。

#### Bean

我们提供的接口分为输入参数（request）和输出参数（response），输入参数为客户端请求时传入，输出参数为后端接口返回的数据。我们在定义接口时最好将输入参数和输出参数放到
request 和 response 包下，在定义的 Bean 下抽象出 Base 类来，如下代码：

    
    
    package com.lynn.common.model;
    
    public abstract class BaseModel {
    
        private Long id;
    
        public Long getId() {
            return id;
        }
    
        public void setId(Long id) {
            this.id = id;
        }
    }
    
    
    
    package com.lynn.common.model.response;
    
    import com.lynn.common.model.BaseModel;
    
    public abstract class BaseResponse extends BaseModel{
    
    }
    
    
    
    package com.lynn.common.model.request;
    
    public abstract class BaseRequest {
    }
    

#### Service和Controller

同样地，我们也可以定义出 BaseService 和 BaseController，在 BaseService 中实现公共方法：

    
    
    package com.lynn.common.service;
    
    import com.lynn.common.encryption.Algorithm;
    import com.lynn.common.encryption.MessageDigestUtils;
    
    public abstract class BaseService {
    
        /**
         * 密码加密算法
         * @param password
         * @return
         */
        protected String encryptPassword(String password){
            return MessageDigestUtils.encrypt(password, Algorithm.SHA1);
        }
    
        /**
         * 生成API鉴权的Token
         * @param mobile
         * @param password
         * @return
         */
        protected String getToken(String mobile,String password){
            return MessageDigestUtils.encrypt(mobile+password, Algorithm.SHA1);
        }
    }
    

我们也可以在 BaseController 里写公共方法：

    
    
    package com.lynn.common.controller;
    
    import org.springframework.util.Assert;
    import org.springframework.validation.BindingResult;
    import org.springframework.validation.FieldError;
    
    import java.util.List;
    
    public abstract class BaseController {
    
        /**
         * 接口输入参数合法性校验
         *
         * @param result
         */
        protected void validate(BindingResult result){
            if(result.hasFieldErrors()){
                List<FieldError> errorList = result.getFieldErrors();
                errorList.stream().forEach(item -> Assert.isTrue(false,item.getDefaultMessage()));
            }
        }
    }
    

接下来，我们就可以来实现具体的业务了。

### 用户模块

根据第14课提供的原型设计图，我们可以分析出，用户模块大概有如下几个接口：

  * 登录
  * 注册
  * 获得用户评论

接下来我们来实现具体的业务（以登录为例），首先是 Bean：

    
    
    package com.lynn.user.model.bean;
    
    import com.lynn.common.model.BaseModel;
    
    public class UserBean extends BaseModel{
    
        private String mobile;
    
        private String password;
    
        public String getMobile() {
            return mobile;
        }
    
        public void setMobile(String mobile) {
            this.mobile = mobile;
        }
    
        public String getPassword() {
            return password;
        }
    
        public void setPassword(String password) {
            this.password = password;
        }
    }
    package com.lynn.user.model.request;
    
    import org.hibernate.validator.constraints.NotEmpty;
    
    public class LoginRequest {
    
        @NotEmpty
        private String mobile;
    
        @NotEmpty
        private String password;
    
        public String getMobile() {
            return mobile;
        }
    
        public void setMobile(String mobile) {
            this.mobile = mobile;
        }
    
        public String getPassword() {
            return password;
        }
    
        public void setPassword(String password) {
            this.password = password;
        }
    }
    

其次是 Mapper（框架采用 Mybatis 的注解方式)：

    
    
    package com.lynn.user.mapper;
    
    import com.lynn.user.model.bean.UserBean;
    import org.apache.ibatis.annotations.Mapper;
    import org.apache.ibatis.annotations.Select;
    
    import java.util.List;
    
    @Mapper
    public interface UserMapper {
    
        @Select("select id,mobile,password from news_user where mobile = #{mobile} and password = #{password}")
        List<UserBean> selectUser(String mobile,String password);
    }
    

然后是 Service（具体的业务实现）：

    
    
    package com.lynn.user.service;
    
    import com.lynn.common.result.Code;
    import com.lynn.common.result.SingleResult;
    import com.lynn.common.service.BaseService;
    import com.lynn.user.mapper.UserMapper;
    import com.lynn.user.model.bean.UserBean;
    import com.lynn.user.model.request.LoginRequest;
    import com.lynn.user.model.response.TokenResponse;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Transactional;
    
    import java.util.List;
    
    @Transactional(rollbackFor = Exception.class)
    @Service
    public class UserService extends BaseService{
    
        @Autowired
        private UserMapper userMapper;
    
        public SingleResult<TokenResponse> login(LoginRequest request){
            List<UserBean> userList = userMapper.selectUser(request.getMobile(),request.getPassword());
            if(null != userList && userList.size() > 0){
                String token = getToken(request.getMobile(),request.getPassword());
                TokenResponse response = new TokenResponse();
                response.setToken(token);
                return SingleResult.buildSuccess(response);
            }else {
                return SingleResult.buildFailure(Code.ERROR,"手机号或密码输入不正确！");
            }
        }
    

我们写的接口要提供给客户端调用，因此最后还需要添加 Controller：

    
    
    package com.lynn.user.controller;
    
    import com.lynn.common.controller.BaseController;
    import com.lynn.common.result.SingleResult;
    import com.lynn.user.model.request.LoginRequest;
    import com.lynn.user.model.response.TokenResponse;
    import com.lynn.user.service.UserService;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.validation.BindingResult;
    import org.springframework.web.bind.annotation.RequestBody;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    
    import javax.validation.Valid;
    
    @RequestMapping("user")
    @RestController
    public class UserController extends BaseController {
    
        @Autowired
        private UserService userService;
    
        @RequestMapping("login")
        public SingleResult<TokenResponse> login(@Valid @RequestBody LoginRequest request, BindingResult result){
            //必须要调用validate方法才能实现输入参数的合法性校验
            validate(result);
            return userService.login(request);
        }
    }
    

这样一个完整的登录接口就写完了。

为了校验我们写的接口是否有问题可以通过 JUnit 来进行单元测试：

    
    
    package com.lynn.user.test;
    
    import com.lynn.user.Application;
    import com.lynn.user.model.request.LoginRequest;
    import com.lynn.user.service.UserService;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
    
    @RunWith(SpringJUnit4ClassRunner.class)
    @SpringBootTest(classes = Application.class)
    public class TestDB {
    
        @Autowired
        private UserService userService;
    
        @Test
        public void test(){
            try {
                LoginRequest request = new LoginRequest();
                request.setMobile("13800138000");
                request.setPassword("1");
                System.out.println(userService.login(request));
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }
    

### 总结

在定义接口之前首先应该分析该接口的输入参数和输出参数，分别定义到 request 和 response 里，在 request 里添加校验的注解，如
NotNull（不能为 null）、NotEmpty（不能为空）等等。

在定义具体的接口，参数为对应的 request，返回值为 `SingleResult<Response>` 或
`MultiResult<Response>`，根据具体的业务实现具体的逻辑。

最后添加 Controller，就是调用 Service 的代码，方法参数需要加上 `@Valid`，这样参数校验才会生效，在调 Service 之前调用
`validate(BindResult)` 方法会抛出参数不合法的异常。最后，我们通过 JUnit 进行单元测试。

