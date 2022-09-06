### JSON

JSON对象：{ "name": "a", "sex": "man" }

JSON数组：[ { "name": "a", "sex": "man" }, { "name": "b", "sex": "man" } ]

JSON字符串："{ \"name\": \"a\", \"sex\": \"man\" }"

JSON字符串数组："[ { \"name\": \"a\", \"sex\": \"man\" }, { \"name\": \"b\", \"sex\": \"man\" } ]"



#### FastJson

> 阿里巴巴开源

JSONObject当作`Map<String, Object>` ，JSONArray当作`List<Object>`

```java
MyObj obj = JSONObject.parseObject(JSON_OBJ_STR, MyObj.class); // JSON字符串 => Java对象
String jsonStr = JSONObject.toJSONString(obj);                 // Java对象 => JSON字符串

JSONObject jsonObject = JSONObject.parseObject(JSON_OBJ_STR);  // JSON字符串 => JSON对象
String jsonStr = JSONObject.toJSONString(jsonObject);          // JSON对象 => JSON字符串

MyObj obj = JSONObject.parseObject(jsonStr, MyObj.class);      // JSON对象 ===> Java对象
JSONObject jsonObject = (JSONObject) JSONObject.toJSON(obj);   // Java对象 => JSON对象

// String str = "[{\"id\":1000,\"name\":\"Jobs\"}, {...}]";
// 转换结果会保留泛型信息
List<Map<String, Object>> list = 
    JSON.parseObject(str, new TypeReference<List<Map<String, Object>>>(){});
```

把`parseObject()`变成`parseArray()`就可以转换JSON数组，结果为List类型

`@JSONField`：
ordinal：序列化时属性的顺序
name：序列化时指定属性名称
format：格式化
serialize：是否序列化
deserialize：是否反序列化



#### Jackson

> SpringBoot自带

```java
ObjectMapper mapper = new ObjectMapper();

MyObj obj = mapper.readValue(jsonStr, MyObj.class);    // JSON字符串 => Java对象
String jsonStr = mapper.writeValueAsString(obj);       // Java对象 => JSON字符串

JsonNode jsonNode = mapper.readTree(jsonStr);          // JSON字符串 => JSON对象
String jsonStr = mapper.writeValueAsString(jsonNode);  // JSON对象 => JSON字符串

MyObj obj = mapper.readValue(jsonStr, MyObj.class);    // JSON对象 ===> Java对象
JsonNode node = mapper.valueToTree(obj);               // Java对象 => JSON对象
```

`@JsonIgnore`：（属性）序列化时忽略此属性

`@JsonIgnoreProperties({"age", "birthTime"})`：（类）序列化时忽略指定属性

`@JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")`：（属性、getter方法）序列化时格式化日期

`@JsonInclude(value = JsonInclude.Include.NON_EMPTY)`：（属性）序列化时不输出""和null

`@JsonSerialize`：（类）随意格式化类 <https://blog.csdn.net/TuxedoLinux/article/details/80782879>

`@JsonProperty(value = "isParent")`：（属性、getter方法）序列化时指定属性名称



序列化时对Java8的日期类进行格式化的设置

```java
ObjectMapper objectMapper = new ObjectMapper();
JavaTimeModule javaTimeModule = new JavaTimeModule();
objectMapper.registerModule(javaTimeModule);
result = objectMapper.writeValueAsString(person);
```

