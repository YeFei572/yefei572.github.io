---
layout: post
tags: 后端
categories: 技术分享
title:  "springdoc-openapi-javadoc无侵入注解模式生成接口文档使用方法.md"
---
> 随着技术升级，现在的接口文档越来越叼了，经历过了手写接口文档、注解式文档，现在终于迎来了无侵入式文档哩~ 不废话，开整！

### 一、老规矩，引入依赖
```xml
<properties>
    <springdoc.version>1.6.15</springdoc.version>
</properties>
<dependencies>
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-webmvc-core</artifactId>
        <version>${springdoc.version}</version>
    </dependency>

    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-javadoc</artifactId>
        <version>${springdoc.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.9.0</version>
            <configuration>
                <source>${java.version}</source>
                <target>${java.version}</target>
                <encoding>${project.build.sourceEncoding}</encoding>
                <annotationProcessorPaths>
                    <path>
                        <groupId>com.github.therapi</groupId>
                        <artifactId>therapi-runtime-javadoc-scribe</artifactId>
                        <version>0.15.0</version>
                    </path>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>${lombok.version}</version>
                    </path>
                    <path>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-configuration-processor</artifactId>
                        <version>${spring-boot.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 二、开始写配置信息

1、`properties`文件
```java
package com.example.tlogdemo.config.properties;

import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.Paths;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.tags.Tag;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.NestedConfigurationProperty;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * swagger 配置属性
 *
 * @author keppel
 */
@Data
@Component
@ConfigurationProperties(prefix = "springdoc")
public class SpringDocProperties {

    /**
     * 文档基本信息
     */
    @NestedConfigurationProperty
    private InfoProperties info = new InfoProperties();

    /**
     * 扩展文档地址
     */
    @NestedConfigurationProperty
    private ExternalDocumentation externalDocs;

    /**
     * 标签
     */
    private List<Tag> tags = null;

    /**
     * 路径
     */
    @NestedConfigurationProperty
    private Paths paths = null;

    /**
     * 组件
     */
    @NestedConfigurationProperty
    private Components components = null;

    /**
     * <p>
     * 文档的基础属性信息
     * </p>
     *
     * @see io.swagger.v3.oas.models.info.Info
     *
     * 为了 springboot 自动生产配置提示信息，所以这里复制一个类出来
     */
    @Data
    public static class InfoProperties {

        /**
         * 标题
         */
        private String title = null;

        /**
         * 描述
         */
        private String description = null;

        /**
         * 联系人信息
         */
        @NestedConfigurationProperty
        private Contact contact = null;

        /**
         * 许可证
         */
        @NestedConfigurationProperty
        private License license = null;

        /**
         * 版本
         */
        private String version = null;

    }

}
```

2、配置相关：
```java
package com.example.tlogdemo.config;

import com.example.tlogdemo.config.properties.SpringDocProperties;
import com.example.tlogdemo.handler.OpenApiHandler;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.Paths;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import lombok.RequiredArgsConstructor;
import org.apache.commons.lang3.StringUtils;
import org.springdoc.core.*;
import org.springdoc.core.customizers.OpenApiBuilderCustomizer;
import org.springdoc.core.customizers.OpenApiCustomiser;
import org.springdoc.core.customizers.ServerBaseUrlCustomizer;
import org.springdoc.core.providers.JavadocProvider;
import org.springframework.boot.autoconfigure.AutoConfigureBefore;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.autoconfigure.web.ServerProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.Set;

/**
 * Swagger 文档配置
 *
 * @author keppel
 */
@RequiredArgsConstructor
@Configuration
@AutoConfigureBefore(SpringDocConfiguration.class)
@ConditionalOnProperty(name = "springdoc.api-docs.enabled", havingValue = "true", matchIfMissing = true)
public class SpringDocConfig {

    private final ServerProperties serverProperties;

    @Bean
    @ConditionalOnMissingBean(OpenAPI.class)
    public OpenAPI openApi(SpringDocProperties properties) {
        OpenAPI openApi = new OpenAPI();
        // 文档基本信息
        SpringDocProperties.InfoProperties infoProperties = properties.getInfo();
        Info info = convertInfo(infoProperties);
        openApi.info(info);
        // 扩展文档信息
        openApi.externalDocs(properties.getExternalDocs());
        openApi.tags(properties.getTags());
        openApi.paths(properties.getPaths());
        openApi.components(properties.getComponents());
        Set<String> keySet = properties.getComponents().getSecuritySchemes().keySet();
        List<SecurityRequirement> list = new ArrayList<>();
        SecurityRequirement securityRequirement = new SecurityRequirement();
        keySet.forEach(securityRequirement::addList);
        list.add(securityRequirement);
        openApi.security(list);

        return openApi;
    }

    private Info convertInfo(SpringDocProperties.InfoProperties infoProperties) {
        Info info = new Info();
        info.setTitle(infoProperties.getTitle());
        info.setDescription(infoProperties.getDescription());
        info.setContact(infoProperties.getContact());
        info.setLicense(infoProperties.getLicense());
        info.setVersion(infoProperties.getVersion());
        return info;
    }

    /**
     * 自定义 openapi 处理器
     */
    @Bean
    public OpenAPIService openApiBuilder(Optional<OpenAPI> openAPI,
                                         SecurityService securityParser,
                                         SpringDocConfigProperties springDocConfigProperties, PropertyResolverUtils propertyResolverUtils,
                                         Optional<List<OpenApiBuilderCustomizer>> openApiBuilderCustomisers,
                                         Optional<List<ServerBaseUrlCustomizer>> serverBaseUrlCustomisers, Optional<JavadocProvider> javadocProvider) {
        return new OpenApiHandler(openAPI, securityParser, springDocConfigProperties, propertyResolverUtils, openApiBuilderCustomisers, serverBaseUrlCustomisers, javadocProvider);
    }

    /**
     * 对已经生成好的 OpenApi 进行自定义操作
     */
    @Bean
    public OpenApiCustomiser openApiCustomiser() {
        String contextPath = serverProperties.getServlet().getContextPath();
        String finalContextPath;
        if (StringUtils.isBlank(contextPath) || "/".equals(contextPath)) {
            finalContextPath = "";
        } else {
            finalContextPath = contextPath;
        }
        // 对所有路径增加前置上下文路径
        return openApi -> {
            Paths oldPaths = openApi.getPaths();
            if (oldPaths instanceof PlusPaths) {
                return;
            }
            PlusPaths newPaths = new PlusPaths();
            oldPaths.forEach((k,v) -> newPaths.addPathItem(finalContextPath + k, v));
            openApi.setPaths(newPaths);
        };
    }

    /**
     * 单独使用一个类便于判断 解决springdoc路径拼接重复问题
     *
     * @author Lion Li
     */
    static class PlusPaths extends Paths {

        public PlusPaths() {
            super();
        }
    }

}

```

### 三、配置`application.yml`
```yaml
springdoc:
  api-docs:
    # 是否开启接口文档
    enabled: true
  info:
    title: '测试后台管理系统——接口文档'
    description: '这是一段描述信息'
    version: '版本号:1.0'
    contact:
      name: keppel
      email: keppelfei@gmail.com
      url: https://www.v2ss.cn
  components:
    # 鉴权方式配置
    security-schemes:
      apiKey:
        type: APIKEY
        in: HEADER
        name: cim-plus
  group-configs:
    - group: 1.测试模块
      packages-to-scan: com.example.tlogdemo
```

### 四、最终访问导出json文件
启动项目后，访问地址：`http://localhost:8080/v3/api-docs` 浏览器中访问可以直接导出对应的json文件，也可以使用模块访问。在`application.yml`中的`group-configs`下面的数组加对应的模块就可以分模块导出，如下：`http://localhost:8080/v3/api-docs/1.测试模块`

最后可以使用各种支持`OpenApi`的相关接口的相关的工具进行导入就可以从容使用了。
