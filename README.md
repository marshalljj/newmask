# 100行代码实现配置化脱敏脱敏

## introduction
使用方式如下，在返回值类型中添加注解如下即可。

```java
@Masks({
    @Mask(jsonPath="$.idCard", from=4, to=10),//脱敏4-8位
    @Mask(jsonPath="$.mobile", from=8),//脱敏8-最后
    @Mask(jsonPath="$.name", to=1)//脱敏前四位
})
public class User {
    private int id;
    private String idCard;
    private String mobile;
    
    //getter,setter
}
```



## design
### 关键概念 HttpMessageConverter
springMVC自带的扩展接口，用于将Controller返回值序列化。
### 思路
自定义HttpMessageConverter，做如下事情：
1. 先将返回值转换成json格式。
2. 再通过JsonPath对注解中指定json路径进行脱敏处理。
3. 脱敏后的jsonString写入到输出流。

## implement
先上代码：
```java
public class MaskMessageConverter implements HttpMessageConverter<Object> {

    @Autowired
    private ObjectMapper objectMapper ;

    private static final Set<MediaType> SUPPORT_MEDIA = Sets.newHashSet(MediaType.APPLICATION_JSON, MediaType.APPLICATION_JSON_UTF8);


    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        return false;
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        return clazz.isAnnotationPresent(Masks.class) && SUPPORT_MEDIA.contains(mediaType);
    }

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return Lists.newArrayList(SUPPORT_MEDIA);
    }

    @Override
    public Object read(Class<?> clazz, HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException {
        throw new UnsupportedOperationException();
    }

    @Override
    public void write(Object o, MediaType contentType, HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException {

        //先将返回值转换成json格式。
        String json = null;
        try {
            json = objectMapper.writeValueAsString(o);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }

        //通过JsonPath对注解中指定json路径进行脱敏处理。
        DocumentContext documentContext = JsonPath.parse(json);
        Masks masks = o.getClass().getAnnotation(Masks.class);
        for (Mask item : masks.value()) {
            documentContext.map(item.jsonPath(), new MapFunction() {
                @Override
                public Object map(Object currentValue, Configuration configuration) {
                    return MaskHelper.mask((String)currentValue, item.from(), item.to());
                }
            });
        }
        String string = documentContext.jsonString();
        
        //脱敏后的jsonString写入到输出流
        outputMessage.getBody().write(string.getBytes());

    }
}

```



