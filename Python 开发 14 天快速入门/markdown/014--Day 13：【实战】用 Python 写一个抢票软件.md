本文将介绍如何用 Python 语言实现 12306 自动预定列车票，也就是坊间常说的“抢票”，但个人觉得，这不算是“抢”，只不过是一定程度的自动化。

![enter image description
here](http://images.gitbook.cn/8d791570-4d2d-11e8-a87b-c51afd12dea6)

### 总体设计

所谓抢票软件，本质上就是基于浏览器驱动，实现登录、预定、确认信息的自动化。购买列车票涉及4个网页，相应的基本流程如下：

![enter image description
here](http://images.gitbook.cn/ca208ed0-4e7d-11e8-bf5a-d5b4a68c7aca)

  1. 登录：输入用户名、密码，识别验证码，点击“登录”；
  2. 基本信息填写：出发地，目的地，出发日期，车票类型(普通或学生)，车次类型选择，点击“查询”，如果目标车次尚有余票则点击“预定”，否则再次点击查询……；
  3. 订单信息填写：乘车人选择，席别选择，票种选择，点击“提交订单”；
  4. 订单确认：选择座位位置，点击“确认”。

### 详细设计

总体设计理清了抢票的主要步骤，进一步需要明确每个步骤中需要注意的问题。

**1\. 登录**

登录过程中，自动输入用户名和密码比较简单，难点在于识别验证码。截至目前，各种自动识别验证码的方案准确率都不高，因此，本文采用“人工辅助”识别验证码，即：识别验证码由人工完成，选择图形验证码后点击“登陆”。

![enter image description
here](http://images.gitbook.cn/a08385e0-4e7e-11e8-b8a0-196886eb8964)

**2\. 基本信息填写、查询、预定**

整体上没有难点，但需要注意，出发地和目的地可能有多个车次，每个车次有多种席别，乘车方案可能比较复杂，比如：路途较远的情况下，对于 G
字头、D字头列车，二等座及以上可接受；对于 K
字头、T字头列车，硬卧及以上可接受……。如此，在抢票的时候，需要按优先级轮询各种方案。以杭州->成都为例，有5个车次可选，如下所示：

![enter image description
here](http://images.gitbook.cn/0facb4f0-4e7f-11e8-b8a0-196886eb8964)

**3\. 订单信息填写**

乘车人列表中可能有多个人的信息（如果你曾经帮别人买过车票的话，注册信息会保留），需要选择正确的乘车人、票种和席别，如下例子所示：

![enter image description
here](http://images.gitbook.cn/d97b1f80-4e82-11e8-b8a0-196886eb8964)

**4\. 订单确认**

这一步很简单，点击“确认”即可，毕竟春运期间抢票，一般不会在意位置，能抢到已是幸运。

![enter image description
here](http://images.gitbook.cn/97cfa410-4e83-11e8-bf5a-d5b4a68c7aca)

### 准备工作

根据总体设计，可以将抢票程序规划为5个主要函数：

  * `__init__()`：初始化
  * `login_proc()`：登录模块
  * `filling_proc()`：基本信息填写模块
  * `booking_proc()`：查询、预订、订单信息填写模块
  * `confirm_proc()`：订单确认模块

**1\. 浏览器驱动**

本文介绍的抢票软件基于 Chrome 浏览器，因此，需要下载与之版本匹配的驱动
chromedriver（附：[下载网址](http://chromedriver.storage.googleapis.com/index.html)）。注意与自己的
Chrome 版本对应，步骤如下：

  * 首先，查看 Chrome 的版本，选择“设置” -> “关于 Chrome”，如下图版本为 66.x。

![enter image description
here](http://images.gitbook.cn/7c919520-4e86-11e8-839d-8b701e7cdd86)

  * 然后，进入 chromedriver 下载网址，根据 notes.txt 文件提供的信息选择正确版本的驱动。如下图，chromedriver2.38 支持 Chrome 版本为65-67：

![enter image description
here](http://images.gitbook.cn/b0759440-4e86-11e8-bf5a-d5b4a68c7aca)

![enter image description
here](http://images.gitbook.cn/bb5cb780-4e86-11e8-80cf-2f3d26992d21)

**2\. Selenium 模块准备**

Selenium 是一个用于 Web 应用程序自动化测试的工具，可直接运行在浏览器中，模拟真实用户操作。支持的浏览器包括 IE、Mozilla
Firefox、Safari、Chrome、Opera
等。由于其功能强大，被广泛应用于网络爬虫的开发，本文将用它作为抢票程序的核心模块（附：[下载及安装方法](https://pypi.org/project/selenium/)）。

**3\. 必要信息准备**

列车购票官网经过数次改革，出发地、目的地、车次、席别等都不是明文，而是以编码表示，因此，需要提前准备好这些信息。信息获取方法：谷歌浏览器打开 12306
官网购票页面，鼠标右键“查看”可以获取到上述信息，以杭州->成都为例：

    
    
        #自定义变量区
        value_fromstation = '%u676D%u5DDE%2CHZH'  # 始发站（杭州）
        value_tostation = '%u6210%u90FD%2CCDW'  # 终点站（成都）
        value_date = '2018-05-10'  # 出发时间
        username=u"username" # 用户名
        password="password" # 密码
        #杭州-成都：车次&席别&预定
        #车次信息字典，数据分别表示车次、一等座ID、二等座ID、无座ID、对应车次的预定按钮ID
        train_info = {"D2222":[['ZY_56000D222251', 'ZE_56000D222251', 'WZ_56000D222251'], 'ticket_56000D222251'],
                  "D2262":[['ZY_56000D226251', 'ZE_56000D226251', 'WZ_56000D226251'], 'ticket_56000D226251']}
    

车次、票种编码，主要网页 URL 是固定的，如下：

    
    
        #车票类型字典，"学生票"和"普通票"对应的ID
        ticket_type_dict = {'student': '//input[@name="sf" and @id="sf1"]',
                                'common': '//input[@name="sf" and @id="sf2"]'}
        #车次类型字典
        train_type_dict = {'T': '//input[@name="cc_type" and @value="T"]',  # 特快
                               'G': '//input[@name="cc_type" and @value="G"]',  # 高铁
                               'D': '//input[@name="cc_type" and @value="D"]',  # 动车
                               'Z': '//input[@name="cc_type" and @value="Z"]'}  # 直达
    
        #登陆页面url
        login_url = 'https://kyfw.12306.cn/otn/login/init'
        #个人信息页面url
        initmy_url = "https://kyfw.12306.cn/otn/index/initMy12306"  
        #订票页面url
        book_url = 'https://kyfw.12306.cn/otn/leftTicket/init'   
        #乘客选择页面url
        confirm_url = 'https://kyfw.12306.cn/otn/confirmPassenger/initDc'
    

### 登陆模块 login_proc() 设计

请参见下面代码：

    
    
    def __init__(self):
            """
            Info:构造函数，创建一个浏览器对象
            """
            print(u"欢迎使用列车订票工具")
            self.driver = webdriver.Chrome(self.driver_path)
            self.driver.implicitly_wait(300)
    
        def login_proc(self):
            """
            Info:登陆过程处理函数，其中图形验证码需要手动选择
            """
            self.driver.get(self.login_url)
            # sign in the user name
            try:
                self.driver.find_element_by_id("username").send_keys(self.username)
                self.driver.find_element_by_id("password").send_keys(self.password)
            except Exception  as err:
                print(u"输入用户名或密码失败!",err)
    
            #点击验证码，人工辅助，目前识别图形验证码比较困难，因此选择人工辅助  
            print(u"请自行选择验证码，点击登陆")
            while True:
                if(self.driver.current_url != self.initmy_url):
                    time.sleep(1)
                else:
                    print('Login finished!')
                    break
    

### 基本信息填写模块 filling_proc() 设计

请参见下面代码：

    
    
    def filling_proc(self,train_type,ticket_type):
            """
            Info:填写起始站，终点站，出发时间，车次类型，车票类型等信息
            """
            print (u'列车类型:', train_type)
            print (u'车票类型:', ticket_type)
    
            # 打开订票网页
            self.driver.get(self.book_url)
            # 选择始发站
            self.driver.add_cookie({"name": "_jc_save_fromStation", "value": self.value_fromstation})
            # 选择终点站
            self.driver.add_cookie({"name": "_jc_save_toStation", "value": self.value_tostation})
            # 选择出发日期
            self.driver.add_cookie({"name": "_jc_save_fromDate", "value": self.value_date})
            self.driver.refresh()
            # 选择车次类型                
            if (train_type == 'T' or train_type == 'G' or train_type == 'D' or train_type == 'Z'):
                self.driver.find_element_by_xpath(self.train_type_dict[train_type]).click()
            else:
                print (u"车次类型异常或未选择!(train_type=%s)" % train_type)
    
            # 选择车票类型
            if (ticket_type == 'student' or ticket_type == 'common'):
                self.driver.find_element_by_xpath(self.ticket_type_dict[ticket_type]).click()
            else:
                print (u"车票类型异常或未选择!(train_type=%s)" % ticket_type)
    

### 查询、预订、订单信息填写模块 booking_proc() 设计

请参见下面代码：

    
    
    def booking_proc(self,refresh_interval=0):
            """
            Info:订票处理过程，循环查询符合条件的车次，如果存在则点击“预定”
            """
            book_ticket_flag = False
            # 循环查询
            while True:
                time.sleep(refresh_interval)
                # 点击“查询”按钮，刷新页面开始查询，查询按钮的ID="query_ticket"
                search_btn = WebDriverWait(self.driver, 10).until(
                    EC.presence_of_element_located((By.XPATH, '//*[@id="query_ticket"]')))
                search_btn.click()
                # 扫描查询结果,根据自定义车次字典train_info提供的信息，逐一查询
                try:
                    for train in self.train_info:
                        print(u"当前查询车次为:"+train)
                        # 根据车次查询对应的席别:商务，一等，二等，无座等
                        seat_list= self.train_info.get(train)
                        for seat in seat_list[0]:
                            ticket_seat_id = '//*[@id="' + seat + '"]'# 席别ID
    
                            tic_tb_item = 'default'
                            # 获取车票数量信息："-"，"无"，"数字"
                            tic_tb_item = WebDriverWait(self.driver, 2).until(
                                EC.presence_of_element_located((By.XPATH, ticket_seat_id)))
                            tic_ava_num = tic_tb_item.text
    
                            # 无票或未开售，则结束当前查询
                            if(tic_ava_num == u'无' or tic_ava_num == u'*'):  
                                continue
                            # 如果车次有票，则点击对应车次的“预定”按钮
                            else:
                                book_ticket_btn = '//*[@id="' + seat_list[1] + '"]/td[13]/a'
                                self.driver.find_element_by_xpath(book_ticket_btn).click()
                                book_ticket_flag = True
                                print(u"开始预定")
                                break
                        if (book_ticket_flag):
                            break
    
                except Exception as err:  
                    print(err)
                    # 网络状态不好的时候，点击查询按钮,可能返回查询结果失败，对此异常可再次点击
                    search_btn.click()
                if (book_ticket_flag):
                    break
    

### 订单确认模块 confirm_proc() 设计

请参见下面代码：

    
    
    def confirm_proc(self):
            """
            Info:点击“预定”之后，需要确认乘客信息和座位信息
            """
            # 判断页面跳是否转至乘客选择页面      
            while True:
                if (self.driver.current_url == self.confirm_url):
                    print (u'页面跳转成功!')
                    break
                else:
                    print (u'等待页面跳转...')
                    time.sleep(1)
            # 乘车人选择:针对乘车人列表多于一人的情况
            print(u"选择乘客")
            while True:
                try:
                    # 选择乘车人列表中的第二个人
                    self.driver.find_element_by_xpath('//*[@id="normalPassenger_1"]').click()
                    break
                except Exception as err:
                    print (u'等待常用联系人列表。。。',err)               
                    time.sleep(0.5)
            try:
                print(u"提交订票信息")
                self.driver.find_element_by_xpath('//*[@id="submitOrder_id"]').click()
                time.sleep(1.5)
                print(u"确认订票信息")
                self.driver.find_element_by_xpath('//*[@id="qr_submit_id"]').click()           
            except Exception as err:
                print (err)      
    

### 完整代码

以预订杭州->成都的列车票为例，完整代码实现如下所示。

    
    
    #-*- coding: utf-8 -*-
    '''
    Created on 2018年1月5日
    
    @author:Shulan Ying
    '''
    from selenium import webdriver
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support.ui import WebDriverWait  # available since 2.4.0
    from selenium.webdriver.support import expected_conditions as EC  # available since 2.26.0
    import time
    
    
    
    class trainBooking:
        """
        Info:用户应根据实际情况自定义以下信息
        """
        #浏览器驱动程序的路径
        driver_path='C:\Program Files (x86)\Google\Chrome\Application\chromedriver'
        #自定义变量区
        value_fromstation = '%u676D%u5DDE%2CHZH'  # 始发站（杭州）
        value_tostation = '%u6210%u90FD%2CCDW'  # 终点站（成都）
        value_date = '2018-05-10'  # 出发时间
        username=u"username" # 用户名
        password=u"password" # 密码
        #杭州-成都：车次&席别&预定
        #车次信息字典，数据分别表示车次、一等座ID、二等座ID、无座ID、对应车次的预定按钮ID
        train_info = {"D2222":[['ZY_56000D222251', 'ZE_56000D222251', 'WZ_56000D222251'], 'ticket_56000D222251'],
                  "D2262":[['ZY_56000D226251', 'ZE_56000D226251', 'WZ_56000D226251'], 'ticket_56000D226251']}
    
        """
        Info:以下信息不需要修改
        """
        #车票类型字典，"学生票"和"普通票"对应的ID
        ticket_type_dict = {'student': '//input[@name="sf" and @id="sf1"]',
                                'common': '//input[@name="sf" and @id="sf2"]'}
        #车次类型字典
        train_type_dict = {'T': '//input[@name="cc_type" and @value="T"]',  # 特快
                               'G': '//input[@name="cc_type" and @value="G"]',  # 高铁
                               'D': '//input[@name="cc_type" and @value="D"]',  # 动车
                               'Z': '//input[@name="cc_type" and @value="Z"]'}  # 直达
    
        #登陆页面url
        login_url = 'https://kyfw.12306.cn/otn/login/init'
        #个人信息页面url
        initmy_url = "https://kyfw.12306.cn/otn/index/initMy12306"  
        #订票页面url
        book_url = 'https://kyfw.12306.cn/otn/leftTicket/init'   
        #乘客选择页面url
        confirm_url = 'https://kyfw.12306.cn/otn/confirmPassenger/initDc'
    
    
        def __init__(self):
            """
            Info:构造函数，创建一个浏览器对象
            """
            print(u"欢迎使用列车订票工具")
            self.driver = webdriver.Chrome(self.driver_path)
            self.driver.implicitly_wait(300)
    
        def login_proc(self):
            """
            Info:登陆过程处理函数，其中图形验证码需要手动选择
            """
            self.driver.get(self.login_url)
            # sign in the user name
            try:
                self.driver.find_element_by_id("username").send_keys(self.username)
                self.driver.find_element_by_id("password").send_keys(self.password)
            except Exception  as err:
                print(u"输入用户名或密码失败!",err)
    
            #点击验证码，人工辅助，目前识别图形验证码比较困难，因此选择人工辅助  
            print(u"请自行选择验证码，点击登陆")
            while True:
                if(self.driver.current_url != self.initmy_url):
                    time.sleep(1)
                else:
                    print('Login finished!')
                    break 
    
        def filling_proc(self,train_type,ticket_type):
            """
            Info:填写起始站，终点站，出发时间，车次类型，车票类型等信息
            """
            print (u'列车类型:', train_type)
            print (u'车票类型:', ticket_type)
    
            # 打开订票网页
            self.driver.get(self.book_url)
            # 选择始发站
            self.driver.add_cookie({"name": "_jc_save_fromStation", "value": self.value_fromstation})
            # 选择终点站
            self.driver.add_cookie({"name": "_jc_save_toStation", "value": self.value_tostation})
            # 选择出发日期
            self.driver.add_cookie({"name": "_jc_save_fromDate", "value": self.value_date})
            self.driver.refresh()
            # 选择车次类型                
            if (train_type == 'T' or train_type == 'G' or train_type == 'D' or train_type == 'Z'):
                self.driver.find_element_by_xpath(self.train_type_dict[train_type]).click()
            else:
                print (u"车次类型异常或未选择!(train_type=%s)" % train_type)
    
            # 选择车票类型
            if (ticket_type == 'student' or ticket_type == 'common'):
                self.driver.find_element_by_xpath(self.ticket_type_dict[ticket_type]).click()
            else:
                print (u"车票类型异常或未选择!(train_type=%s)" % ticket_type)
    
    
        def booking_proc(self,refresh_interval=0):
            """
            Info:订票处理过程，循环查询符合条件的车次，如果存在则点击“预定”
            """
            book_ticket_flag = False
            # 循环查询
            while True:
                time.sleep(refresh_interval)
                # 点击“查询”按钮，刷新页面开始查询，查询按钮的ID="query_ticket"
                search_btn = WebDriverWait(self.driver, 10).until(
                    EC.presence_of_element_located((By.XPATH, '//*[@id="query_ticket"]')))
                search_btn.click()
                # 扫描查询结果,根据自定义车次字典train_info提供的信息，逐一查询
                try:
                    for train in self.train_info:
                        print(u"当前查询车次为:"+train)
                        # 根据车次查询对应的席别:商务，一等，二等，无座等
                        seat_list= self.train_info.get(train)
                        for seat in seat_list[0]:
                            ticket_seat_id = '//*[@id="' + seat + '"]'# 席别ID
    
                            tic_tb_item = 'default'
                            # 获取车票数量信息："-"，"无"，"数字"
                            tic_tb_item = WebDriverWait(self.driver, 2).until(
                                EC.presence_of_element_located((By.XPATH, ticket_seat_id)))
                            tic_ava_num = tic_tb_item.text
    
                            # 无票或未开售，则结束当前查询
                            if(tic_ava_num == u'无' or tic_ava_num == u'*'):  
                                continue
                            # 如果车次有票，则点击对应车次的“预定”按钮
                            else:
                                book_ticket_btn = '//*[@id="' + seat_list[1] + '"]/td[13]/a'
                                self.driver.find_element_by_xpath(book_ticket_btn).click()
                                book_ticket_flag = True
                                print(u"开始预定")
                                break
                        if (book_ticket_flag):
                            break
    
                except Exception as err:  
                    print(err)
                    # 网络状态不好的时候，点击查询按钮,可能返回查询结果失败，对此异常可再次点击
                    search_btn.click()
                if (book_ticket_flag):
                    break
    
        def confirm_proc(self):
            """
            Info:点击“预定”之后，需要确认乘客信息和座位信息
            """
            # 判断页面跳是否转至乘客选择页面      
            while True:
                if (self.driver.current_url == self.confirm_url):
                    print (u'页面跳转成功!')
                    break
                else:
                    print (u'等待页面跳转...')
                    time.sleep(1)
            # 乘车人选择:针对乘车人列表多于一人的情况
            print(u"选择乘客")
            while True:
                try:
                    # 选择乘车人列表中的第二个人
                    self.driver.find_element_by_xpath('//*[@id="normalPassenger_1"]').click()
                    break
                except Exception as err:
                    print (u'等待常用联系人列表。。。',err)               
                    time.sleep(0.5)
            try:
                print(u"提交订票信息")
                self.driver.find_element_by_xpath('//*[@id="submitOrder_id"]').click()
                time.sleep(1.5)
                print(u"确认订票信息")
                self.driver.find_element_by_xpath('//*[@id="qr_submit_id"]').click()           
            except Exception as err:
                print (err)      
    
    
    if __name__ == '__main__':
        leave_date = '2018-05-10'
        train_type = 'D'
        ticket_type = 'common'
        refresh_interval = 0.1
        booker=trainBooking()
        booker.login_proc()
        booker.filling_proc(train_type, ticket_type)
        booker.booking_proc(refresh_interval)
        booker.confirm_proc()
    

