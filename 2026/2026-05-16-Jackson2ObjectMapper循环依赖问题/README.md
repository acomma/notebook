## 问题

在我们项目里有这样一份代码片段：

```java
@Configuration
public class FooWebMvcConfigurer implements WebMvcConfigurer {
    @Autowired
    private Jackson2ObjectMapperBuilder jackson2ObjectMapperBuilder;

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        ObjectMapper objectMapper = jackson2ObjectMapperBuilder.build();

        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter(objectMapper);
        converter.setSupportedMediaTypes(List.of(new MediaType("application", "json")));

        converters.add(converter);
    }

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        Jackson2ObjectMapperBuilderCustomizer customizer = new Jackson2ObjectMapperBuilderCustomizer() {
            @Override
            public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
                // 什么也不做
            }
        };
        return customizer;
    }
}

@RestControllerAdvice(basePackages = "com.example.foo")
public class FooResponseBodyAdvice implements ResponseBodyAdvice<Object> {
    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        return body;
    }
}
```

这份代码会导致在启动时提示：

```
***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

   fooResponseBodyAdvice (field private com.fasterxml.jackson.databind.ObjectMapper com.example.foo.FooResponseBodyAdvice.objectMapper)
      ↓
   jacksonObjectMapper defined in class path resource [org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration$JacksonObjectMapperConfiguration.class]
┌─────┐
|  jacksonObjectMapperBuilder defined in class path resource [org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration$JacksonObjectMapperBuilderConfiguration.class]
↑     ↓
|  fooWebMvcConfigurer (field private org.springframework.http.converter.json.Jackson2ObjectMapperBuilder com.example.foo.FooWebMvcConfigurer.jackson2ObjectMapperBuilder)
└─────┘


Action:

Despite circular references being allowed, the dependency cycle between beans could not be broken. Update your application to remove the dependency cycle.
```

## 分析

先来看一个简单的只有一个类导致循环依赖的例子：

```java
@Configuration
public class FooConfiguration {
    @Autowired
    private Jackson2ObjectMapperBuilder jackson2ObjectMapperBuilder;

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        Jackson2ObjectMapperBuilderCustomizer customizer = new Jackson2ObjectMapperBuilderCustomizer() {
            @Override
            public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
                // 什么也不做
            }
        };
        return customizer;
    }
}
```

在启动时提示：

```
***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  fooConfiguration (field private org.springframework.http.converter.json.Jackson2ObjectMapperBuilder com.example.foo.FooConfiguration.jackson2ObjectMapperBuilder)
↑     ↓
|  jacksonObjectMapperBuilder defined in class path resource [org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration$JacksonObjectMapperBuilderConfiguration.class]
└─────┘


Action:

Relying upon circular references is discouraged and they are prohibited by default. Update your application to remove the dependency cycle between beans. As a last resort, it may be possible to break the cycle automatically by setting spring.main.allow-circular-references to true.
```

从失败信息中可以知道 `fooConfiguration` 依赖 `jacksonObjectMapperBuilder`，而 `jacksonObjectMapperBuilder` 依赖 `fooConfiguration`，从而形成循环依赖。根据失败信息提示查看 `Jackson2ObjectMapperBuilder` 源码

```java
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@ConditionalOnMissingBean
Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder(ApplicationContext applicationContext,
        List<Jackson2ObjectMapperBuilderCustomizer> customizers) {
    Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
    builder.applicationContext(applicationContext);
    customize(builder, customizers);
    return builder;
}
```

在创建 `Jackson2ObjectMapperBuilder` 时候会依赖所有的 `Jackson2ObjectMapperBuilderCustomizer`，而 `FooConfiguration` 恰好定义了一个，从而形成循环依赖。这个循环依赖可以通过以下方式之一解决：

1. 配置 `spring.main.allow-circular-references=true`
2. 在 `@Autowired` 的地方加上 `@Lazy`
3. 使用 `ObjectProvider`，比如 `public FooConfiguration(ObjectProvider<Jackson2ObjectMapperBuilder> provider) {}`

对问题中的例子来说只有 `@Lazy` 是有效的，即使方式 1 是有效的也不推荐，它只是掩盖了问题。根本性的解决办法是“定义纯洁的配置类”，即在独立的配置类中定义 `Jackson2ObjectMapperBuilderCustomizer`

```java
@Configuration
public class JacksonConfiguration {
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        Jackson2ObjectMapperBuilderCustomizer customizer = new Jackson2ObjectMapperBuilderCustomizer() {
            @Override
            public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
                // 什么也不做
            }
        };
        return customizer;
    }
}
```
