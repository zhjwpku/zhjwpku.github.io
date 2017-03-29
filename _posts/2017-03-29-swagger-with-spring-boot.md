---
layout: post
title: Using Swagger with Spring Boot
date: 2017-03-29 19:00:00 +0800
tags:
- swagger
---

[Swagger][swagger] 框架核心的框架包含三个核心工具：[Swagger Editor][se]、[Swagger Codegen][sc] 和 [Swagger UI][su]。本文介绍如何使用这三个工具来快速构建 Spring Boot Server。

Editor 用于编辑 Swagger 规格文档，并实时生成对应的 RESTful 接口文档。

Codegen 用于将设计好的 Swagger 规格文档生成不同框架的代码，包括服务端和客户端。

UI 用于将 Swagger 规格文档进行显示以及与 API 资源进行交互。

那这三个工具应该如何使用可能困扰着一些刚接触 Swagger 的开发人员，本文简单介绍一下如何将 Swagger 用于 Spring Boot 框架。

首先，Editor 和 Codegen 这两个工具个人认为并不需要自己安装，直接在 Swagger 的 [Online Editor][oe] 进行 API 的设计和编辑，设计好之后使用界面上侧的 `Generate Server` 选择相应的框架进行生成，如图：

![Codegen](/assets/201703/swagger-codegen.png)

将生成的包解压，并将*src/main/resources/application.properties* 中的 `server.port` 改为一个空闲的端口号（这里假设为8181）。使用 maven 生成可执行 jar 包。

{% highlight shell %}
→ ~/work/myprojects/spring-server $ mvn package
→ ~/work/myprojects/spring-server $ cd target/
# 运行 Spring Boot 程序
→ ~/work/myprojects/spring-server/target $ java -jar swagger-spring-1.0.0.jar
{% endhighlight %}

浏览器访问 `http://localhost:8181/v1/swagger-ui.html` 即可使用 Swagger UI 来消费你的 API 了, 当然，对于任何访问，这个生成的服务器并没有做任何处理，仅仅返回一个状态为 200 的空值，这就是你需要发威的地方了。

代码地址：[spring-server][spring-server]。

我们来看代码，代码结构：

{% highlight shell %}
→ ~/work/myprojects/spring-server (master) $ tree
.
├── README.md
├── pom.xml
├── src
│   └── main
│       ├── java
│       │   └── io
│       │       └── swagger
│       │           ├── RFC3339DateFormat.java
│       │           ├── Swagger2SpringBoot.java                 # Spring Boot 启动程序
│       │           ├── api                                     # 该文件下为定义的接口和实现
│       │           │   ├── ApiException.java
│       │           │   ├── ApiOriginFilter.java
│       │           │   ├── ApiResponseMessage.java
│       │           │   ├── EstimatesApi.java                   # Api结尾的为Interface
│       │           │   ├── EstimatesApiController.java         # Controller结尾的为Implementation
│       │           │   ├── HistoryApi.java
│       │           │   ├── HistoryApiController.java
│       │           │   ├── MeApi.java
│       │           │   ├── MeApiController.java
│       │           │   ├── NotFoundException.java
│       │           │   ├── ProductsApi.java
│       │           │   └── ProductsApiController.java
│       │           ├── configuration
│       │           │   ├── HomeController.java
│       │           │   └── SwaggerDocumentationConfig.java     # Swagger 文档配置
│       │           └── model                                   # 定义的model 
│       │               ├── Activities.java
│       │               ├── Activity.java
│       │               ├── Error.java
│       │               ├── PriceEstimate.java
│       │               ├── Product.java
│       │               └── Profile.java
│       └── resources
│           └── application.properties
└── swagger-spring.iml
{% endhighlight %}

**Spring Boot 启动程序: Swagger2SpringBoot.java**

{% highlight java %}
package io.swagger;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.ExitCodeGenerator;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;

import springfox.documentation.swagger2.annotations.EnableSwagger2;

@SpringBootApplication
@EnableSwagger2         // 用于生成Swagger文档
@ComponentScan(basePackages = "io.swagger")
public class Swagger2SpringBoot implements CommandLineRunner {

    @Override
    public void run(String... arg0) throws Exception {
        if (arg0.length > 0 && arg0[0].equals("exitcode")) {
            throw new ExitException();
        }
    }

    public static void main(String[] args) throws Exception {
        new SpringApplication(Swagger2SpringBoot.class).run(args);
    }

    class ExitException extends RuntimeException implements ExitCodeGenerator {
        private static final long serialVersionUID = 1L;

        @Override
        public int getExitCode() {
            return 10;
        }

    }
}
{% endhighlight %}

**Swagger 文档配置**

{% highlight java %}
package io.swagger.configuration;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;

@javax.annotation.Generated(value = "io.swagger.codegen.languages.SpringCodegen", date = "2017-03-28T06:06:39.062Z")

@Configuration
public class SwaggerDocumentationConfig {

    ApiInfo apiInfo() {
        return new ApiInfoBuilder()
            .title("Uber API")
            .description("Move your app forward with the Uber API")
            .license("")
            .licenseUrl("http://unlicense.org")
            .termsOfServiceUrl("")
            .version("1.0.0")
            .contact(new Contact("","", ""))
            .build();
    }

    @Bean
    public Docket customImplementation(){
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                    .apis(RequestHandlerSelectors.basePackage("io.swagger.api"))    // RESTful Controller 所在的目录
                    .build()
                .directModelSubstitute(org.joda.time.LocalDate.class, java.sql.Date.class)
                .directModelSubstitute(org.joda.time.DateTime.class, java.util.Date.class)
                .apiInfo(apiInfo());
    }

}
{% endhighlight %}

可以看出使用了 [springfox][springfox] 来生成 Swagger 规格文档，如果看 [pom.xml][pom] 文件，你会发现生成的代码使用了 springfox-swagger2 和 springfox-swagger-ui，笔者的理解是，springfox-swagger2 用于将程序中的 Swagger annotation 生成 Swagger 规格文档，而 springfox-swagger-ui 则用于将规格文档进行展现并处理用户在 UI 上对 API 资源的操作。更多 Springfox 相关的内容参见 [Springfox Reference Documentation][springfoxdoc]。

**Home Controller**

{% highlight shell %}
package io.swagger.configuration;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * Home redirection to swagger api documentation
 */
@Controller
public class HomeController {
    @RequestMapping(value = "/")    // 将 basePath/重定向到 swagger-ui.html
    public String index() {
        System.out.println("swagger-ui.html");
        return "redirect:swagger-ui.html";
    }
}
{% endhighlight %}

**Spring Annotations**

Swagger 包含了一些特定的注解，可以更好的显示API:

{% highlight shell %}
@API               表示一个开放的API，可以通过description简要描述该API的功能，一般和 Spring 的 @Controller一起。
@ApiOperation      在一个@API下，可有多个@ApiOperation，表示针对该API的CRUD操作。在ApiOperation
                   中可以通过value，notes描述该操作的作用，response描述正常情况下该请求的返回对象类型。
@ApiResponse       在一个ApiOperation下，可以通过ApiResponses描述该API操作可能出现的异常情况。
@ApiParam          描述该API操作接受的参数类型
@ApiModel          描述封装的参数对象与返回的参数对象
@ApiModelProperty  描述ApiModel的属性
{% endhighlight %}


**总结**

有人喜欢将 Swagger 规格文档单独存放在项目中，内嵌一个 Swagger UI, 并将 index.html 中的 url 指向规格文档，如 ref2。笔者认为这种方式不是特别利于维护，假设代码修改了，而忘了修改 Swagger 规格文档，则会造成不一致。本文使用的 Swagger 注解的方式使得文档跟着代码一块走，更利于文档的维护。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Implementing Swagger (OpenAPI specification) with your REST API documentation][ref1]<br>
2 [Swagger: How to Create an API Documentation][ref2]
</span>

[swagger]: http://swagger.io/
[se]: https://github.com/swagger-api/swagger-editor
[sc]: https://github.com/swagger-api/swagger-codegen
[su]: https://github.com/swagger-api/swagger-ui
[oe]: http://editor.swagger.io/#!/
[spring-server]: https://github.com/zhjwpku/spring-boot-server
[springfox]: springfox.documentation.service.ApiInfo
[springfoxdoc]: http://springfox.github.io/springfox/docs/current/
[pom]: https://github.com/zhjwpku/spring-boot-server/blob/master/pom.xml#L47
[ref1]: http://idratherbewriting.com/learnapidoc/pubapis_swagger_intro.html
[ref2]:https://www.youtube.com/watch?v=xggucT_xl5U
