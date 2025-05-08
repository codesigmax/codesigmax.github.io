## 🧩问题描述

为了方便地格式化存储结构化数据，并利用数据库原生的 JSON 操作，我选择将某些字段存储为 JSON 类型。但在实际开发中发现，MyBatis-Plus 自带的 JSON 类型处理器对 PostgreSQL 的支持并不理想，因此我自行实现了一个客制化的 JSON 类型处理器。

处理器绑定到实体类后，插入操作一切正常，数据也能正确写入数据库。但在执行查询时，虽然能成功查出数据，却触发了反射异常。异常消息为 `argument type mismatch` —— 参数类型不匹配。

## 🛠️解决方法

为相关实体类添加无参构造器，或使用Lombok的`@NoArgsConstructor`和`@AllArgsConstructor`注解自动生成构造器即可。

<!-- more -->

---

## 📦相关代码

json类型处理器（省略imports）
```java
/**
 * 强化的PostgreSQL JSON/JSONB类型处理器(专为Map<String, Object>设计)
 * @author 搴芳
 */
@MappedTypes({Map.class})
@MappedJdbcTypes({JdbcType.OTHER})
public class EnhancedJsonbTypeHandler extends BaseTypeHandler<Map<String, Object>> {
    private static final Logger log = LoggerFactory.getLogger(EnhancedJsonbTypeHandler.class);
    private static final ObjectMapper objectMapper = new ObjectMapper();
    private static final TypeReference<Map<String, Object>> TYPE_REF = new TypeReference<>() {
    };

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                    Map<String, Object> parameter,
                                    JdbcType jdbcType) throws SQLException {
        PGobject pgObject = new PGobject();
        pgObject.setType("jsonb");
        try {
            pgObject.setValue(objectMapper.writeValueAsString(parameter));
        } catch (JsonProcessingException e) {
            throw new SQLException("Error converting Map to JSON", e);
        }
        ps.setObject(i, pgObject);
    }

    @Override
    public Map<String, Object> getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return parseValue(rs.getObject(columnName));
    }

    @Override
    public Map<String, Object> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return parseValue(rs.getObject(columnIndex));
    }

    @Override
    public Map<String, Object> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return parseValue(cs.getObject(columnIndex));
    }

    private Map<String, Object> parseValue(Object value) {
        if (value == null) {
            return Collections.emptyMap();
        }

        try {
            // 处理PostgreSQL的PGobject
            if (value instanceof PGobject pgObject) {
                if (pgObject.getValue() == null) {
                    return Collections.emptyMap();
                }
                return objectMapper.readValue(pgObject.getValue(), TYPE_REF);
            }
            // 处理直接字符串
            else if (value instanceof String str) {
                return objectMapper.readValue(str, TYPE_REF);
            }
            // 其他情况尝试直接转换
            else {
                log.warn("Unexpected JSON type: {}", value.getClass().getName());
                return objectMapper.convertValue(value, TYPE_REF);
            }
        } catch (Exception e) {
            log.error("JSON parsing error. Raw value: {} (Type: {})",
                    value, value.getClass().getSimpleName(), e);
            return Collections.emptyMap();
        }
    }
}
```
> 💡由于我的业务中json数据字段众多并且并非全部required，此处将json数据映射到Map中存储

---

## 🔍错误排查

### ✅初步观察

* 插入操作正常，说明类型处理器工作正常；
* 查询能返回数据，但在 ORM 阶段出现反射异常；
* 日志中未出现类型处理器的错误信息。

### 🔧初步调试

* 既然插入正常，暂时排除处理器问题；
* 报错点集中在 ORM 反射过程中；
* 检查实体类配置，未发现明显问题。

### 🧭新的思路

观察实体类相关配置后，并没有发现可能的错误，猜测是框架内部的映射逻辑导致的异常，目前使用的MyBatis-Plus版本是最新的3.5.12，查阅官方文档并没有找到相关的补充。不过目前面对问题有了新的切入点，可以尝试进行新一轮的调试。

框架在进行orm映射时，一般会优先选用无参构造器+Setter方法来进行，除非开发者手动指定使用全参构造器。而MyBatis-Plus内部机制决定使用反射+Setter方法进行orm，首先使用反射调用无参构造来实例化对象，再使用Setter方法进行注入。本次出现异常的查询操作，正是使用了MyBatis-Plus内部的查询方法进行的，问题已经显而易见了。

### ✨最终修复

回到相关的实体类中，由于我们在实体类上使用了Lombok的`@Builder`注解来快速支持建造者模式进行实例化操作，`@Builder`注解生成的Builder使用的是全参构造器来实例化对象，因此还需要手动为实体类添加无参构造器或同时使用`@NoArgsConstructor`和`@AllArgsConstructor`注解来让Lombok自动生成。

验证通过后，查询操作恢复正常。

---

_**文毕致谢 💐**_