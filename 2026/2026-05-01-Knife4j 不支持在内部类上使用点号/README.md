`com.github.xiaoymin:knife4j-openapi2-spring-boot-starter:4.4.0` 内部类上的 `@ApiModel` 注解的 `value` 属性不支持 `.` 分隔符。举个例子下面的内部类

```java
@Data
@ApiModel(value = "Outer")
public class Outer {
    private String name;

    private Inner inner;

    @Data
    @ApiModel(value = "Outer.Inner")
    public static class Inner {
        private String name;
    }
}
```

生成的结果如下所示，注意第 7 行的内容，生成的引用是不完整的，缺少 `#/definitions/` 部分

```json
{
    "definitions": {
        "Outer": {
            "type": "object",
            "properties": {
                "inner": {
                    "$ref": "Outer.Inner",
                    "originalRef": "Outer.Inner"
                },
                "name": {
                    "type": "string"
                }
            },
            "title": "Outer"
        },
        "Outer.Inner": {
            "type": "object",
            "properties": {
                "name": {
                    "type": "string"
                }
            },
            "title": "Outer.Inner"
        }
    }
}
```

而没有使用 `.` 定义 `@ApiModel` 注解的 `value` 属性的内部类，比如使用 `$` 定义的内部类

```java
@Data
@ApiModel(value = "Outer")
public class Outer {
    private String name;

    private Inner inner;

    @Data
    @ApiModel(value = "Outer$Inner")
    public static class Inner {
        private String name;
    }
}
```

生成的结果如下所示，注意第 7 行的内容，生成的引用是完整的

```json
{
    "definitions": {
        "Outer": {
            "type": "object",
            "properties": {
                "inner": {
                    "$ref": "#/definitions/Outer$Inner",
                    "originalRef": "Outer$Inner"
                },
                "name": {
                    "type": "string"
                }
            },
            "title": "Outer"
        },
        "Outer$Inner": {
            "type": "object",
            "properties": {
                "name": {
                    "type": "string"
                }
            },
            "title": "Outer$Inner"
        }
    }
}
```

这与 `springfox-swagger2` 无关，使用 `io.springfox:springfox-swagger2:2.9.2` 和 `io.springfox:springfox-swagger-ui:2.9.2` 测试时无论 `.` 还是 `$` 都是正常的。测试发现除 `.` 以外其他字符也都是正常的，所有这应该是 Knife4j 的问题。还不知道是什么原因引起的。
