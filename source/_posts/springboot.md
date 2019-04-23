---
title: Spring Boot 入门第一篇
date: 2018-10-28 09:55:41
categories: Java
tags: [Spring Boot]
---
很久没有学习Java开发相关知识了，由于现在的工作是Android开发，所以接触Android的东西很多，但是我还是对Java后台情有独钟，明年准备跳槽回到Java开发的工作，现在开始从这个比较流行的技术Spring Boot开始吧。
Spring Boot原以为只是一个J2EE框架，其实不是，它应该是一种微服务框架，通俗的讲Spring Boot就是将我们Spring的开发工作简化了，使开发人员不在对配置文件浪费时间了。
## 入门第一天
### 构建maven项目
1、访问http://start.spring.io/ ,Spring提供自动构建demo项目，可以选择利用maven或者gradle构建项目，我选择的是maven
2、maven项目开发结构一般是

- src/main/java  程序开发以及主程序入口
- src/main/resources 配置文件
- src/test/java  测试程序

至此一个项目就可以跑起来了
### 引入Web模块
1、添加web依赖
```
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
 </dependency>
```
2、编写一个controller
```
@RestController
public class HelloWorldController {
    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }
}
```
@RestController注解作用是输出json格式数据
3、现在运行项目，打开浏览器，输入http://localhost:8080/hello ,就可以看到你的输出了
### Spring Boot单元测试
打开的src/test/下的测试入口，编写简单的http请求来测试；使用mockmvc进行，利用MockMvcResultHandlers.print()打印出执行结果。
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class HelloWorldControlerTests {
    private MockMvc mvc;
    @Before
    public void setUp() throws Exception {
        mvc = MockMvcBuilders.standaloneSetup(new HelloWorldController()).build();
    }
    @Test
    public void getHello() throws Exception {
    mvc.perform(MockMvcRequestBuilders.get("/hello").accept(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andDo(MockMvcResultHandlers.print())
                .andReturn();
    }
}
```