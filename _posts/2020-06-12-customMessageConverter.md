
# 目录

1.  [问题](#org8f29860)
2.  [Spring MVC 如何处理返回值](#orgfd4502d)
    1.  [自定义消息转换器](#org740a90e)

<a id="org8f29860"></a>

# 问题

由于存在一些技术债，需要服务层返回的的json即能支持驼峰（默认支持），又能支持下划线。

Spring MVC配置的RestController默认使用jackson将结果转成驼峰格式的json，所以需要自定义一个在符合某个条件下返回的json转成下划线格式。


<a id="orgfd4502d"></a>

# Spring MVC 如何处理返回值

Spring MVC通过HttpMessageConverter来处理数据流和对象的转换，HttpMessageConverter定义了如下方法：

-   boolean canRead(Class<?> clazz, MediaType mediaType)：是否支持该媒体类型将参数转成对象
-   boolean canWrite(Class<?> clazz, MediaType mediaType)：是否支持该媒体类型将对象输出
-   List<MediaType> getSupportedMediaTypes()：转换器支持的媒体类型
-   T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
    throws IOException, HttpMessageNotReadableException：输入转成对象
-   void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
    throws IOException, HttpMessageNotWritableException：对象输出

Spring MVC根据Content-type的媒体类型来指定流转对象的解析器，Accept的媒体类型来决定对象转成流的转换器。选择对象转成流的转换器的代码如下AbstractMessageConverterMethodProcessor.writeWithMessageConverters：
```java
    
    protected <T> void writeWithMessageConverters(T value, MethodParameter returnType,
        ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
        throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
    
      Object outputValue;
      Class<?> valueType;
      Type declaredType;
    
      if (value instanceof CharSequence) {
        outputValue = value.toString();
        valueType = String.class;
        declaredType = String.class;
      }
      else {
        outputValue = value;
        valueType = getReturnValueType(outputValue, returnType);
        declaredType = getGenericType(returnType);
      }
    
      HttpServletRequest request = inputMessage.getServletRequest();
      // Accept的媒体类型
      List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(request);
      // 转换器中支持媒体类型的转换器
      List<MediaType> producibleMediaTypes = getProducibleMediaTypes(request, valueType, declaredType);
    
      if (outputValue != null && producibleMediaTypes.isEmpty()) {
        throw new IllegalArgumentException("No converter found for return value of type: " + valueType);
      }
      // 能够处理该媒体类型的转换器
      Set<MediaType> compatibleMediaTypes = new LinkedHashSet<MediaType>();
      for (MediaType requestedType : requestedMediaTypes) {
        for (MediaType producibleType : producibleMediaTypes) {
          if (requestedType.isCompatibleWith(producibleType)) {
            compatibleMediaTypes.add(getMostSpecificMediaType(requestedType, producibleType));
          }
        }
      }
      if (compatibleMediaTypes.isEmpty()) {
        if (outputValue != null) {
          throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);
        }
        return;
      }
    
      List<MediaType> mediaTypes = new ArrayList<MediaType>(compatibleMediaTypes);
      MediaType.sortBySpecificityAndQuality(mediaTypes);
      // 最终处理的媒体类型
      MediaType selectedMediaType = null;
      for (MediaType mediaType : mediaTypes) {
        if (mediaType.isConcrete()) {
          selectedMediaType = mediaType;
          break;
        }
        else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {
          selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
          break;
        }
      }
    
      if (selectedMediaType != null) {
        selectedMediaType = selectedMediaType.removeQualityValue();
        // 选择初始化中支持该媒体类型的转换器输出
        for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
          if (messageConverter instanceof GenericHttpMessageConverter) {
            if (((GenericHttpMessageConverter) messageConverter).canWrite(
                declaredType, valueType, selectedMediaType)) {
              outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                  (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                  inputMessage, outputMessage);
              if (outputValue != null) {
                addContentDispositionHeader(inputMessage, outputMessage);
                ((GenericHttpMessageConverter) messageConverter).write(
                    outputValue, declaredType, selectedMediaType, outputMessage);
                if (logger.isDebugEnabled()) {
                  logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                      "\" using [" + messageConverter + "]");
                }
              }
              return;
            }
          }
          else if (messageConverter.canWrite(valueType, selectedMediaType)) {
            outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                inputMessage, outputMessage);
            if (outputValue != null) {
              addContentDispositionHeader(inputMessage, outputMessage);
              ((HttpMessageConverter) messageConverter).write(outputValue, selectedMediaType, outputMessage);
              if (logger.isDebugEnabled()) {
                logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                    "\" using [" + messageConverter + "]");
              }
            }
            return;
          }
        }
      }
    
      if (outputValue != null) {
        throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
      }
    }
```

其中常用的@RestController将对象转json是通过MappingJackson2HttpMessageConverter实现的。不做配置情况下，返回的json格式默认是驼峰样式（只有健是如此，值不做处理）。

Spring MVC项目在初始化阶段就会初始化MappingJackson2HttpMessageConverter，具体代码位于WebMvcConfigurationSupport中：

```java
    /**
     * Provides access to the shared {@link HttpMessageConverter}s used by the
     * {@link RequestMappingHandlerAdapter} and the
     * {@link ExceptionHandlerExceptionResolver}.
     * This method cannot be overridden.
     * Use {@link #configureMessageConverters(List)} instead.
     * Also see {@link #addDefaultHttpMessageConverters(List)} that can be
     * used to add default message converters.
     */
    protected final List<HttpMessageConverter<?>> getMessageConverters() {
      // 如果没有消息转换器则初始化消息转换器
      if (this.messageConverters == null) {
        this.messageConverters = new ArrayList<HttpMessageConverter<?>>();
        // 配置消息转换器。本身没有实现，留给定制初始化消息转换器
        configureMessageConverters(this.messageConverters);
        // 如果依然没有消息转换器，那么添加默认转换器
        if (this.messageConverters.isEmpty()) {
          addDefaultHttpMessageConverters(this.messageConverters);
        }
        // 扩展消息转换器。没有实现，留给定制使用
        extendMessageConverters(this.messageConverters);
      }
      return this.messageConverters;
    }
    
    /**
     * Adds a set of default HttpMessageConverter instances to the given list.
     * Subclasses can call this method from {@link #configureMessageConverters(List)}.
     * @param messageConverters the list to add the default message converters to
     */
    protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
      // 字符串转换工具。如果媒体类型是 text/plain则使用该转换工具，默认字符集是ISO-8859-1
      StringHttpMessageConverter stringConverter = new StringHttpMessageConverter();
      stringConverter.setWriteAcceptCharset(false);
    
      // 字节数组转换工具。如果媒体类型是 application/octet-stream，那么该转换工具
      messageConverters.add(new ByteArrayHttpMessageConverter());
      messageConverters.add(stringConverter);
      messageConverters.add(new ResourceHttpMessageConverter());
      messageConverters.add(new SourceHttpMessageConverter<Source>());
      messageConverters.add(new AllEncompassingFormHttpMessageConverter());
    
      if (romePresent) {
        messageConverters.add(new AtomFeedHttpMessageConverter());
        messageConverters.add(new RssChannelHttpMessageConverter());
      }
    
      if (jackson2XmlPresent) {
        messageConverters.add(new MappingJackson2XmlHttpMessageConverter(
            Jackson2ObjectMapperBuilder.xml().applicationContext(this.applicationContext).build()));
      }
      else if (jaxb2Present) {
        messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
      }
      // 如果WebMvcConfigurationSupport.class.getClassLoader()可以加载到 com.fasterxml.jackson.databind.ObjectMapper 和 com.fasterxml.jackson.core.JsonGenerator 那么启用MappingJackson2HttpMessageConverter转换器。即如果不自定义configureMessageConverters，那么MappingJackson2HttpMessageConverter是默认启动的转换器
      if (jackson2Present) {
        messageConverters.add(new MappingJackson2HttpMessageConverter(
            Jackson2ObjectMapperBuilder.json().applicationContext(this.applicationContext).build()));
      }
      else if (gsonPresent) {
        messageConverters.add(new GsonHttpMessageConverter());
      }
    }
```

如果只是单独希望转成下划线样式，那么可以直接重写extendMessageConverters方法，获取MappingJackson2HttpMessageConverter，然后属性名称命名策略改成下划线方式。 


<a id="org740a90e"></a>

## 自定义消息转换器

由于服务接口众多，并且需要同时支持驼峰和下划线，所以决定自定义一个媒体类型比如：application/underscore，如果调用房指定是Accept:application/underscore，那么有自定义模版消息将结果转成下划线格式的json。

自定义媒体消息：

```java
    public class CustomMediaType extends MediaType {
         public CustomMediaType(String type, String subtype) {
             super(type, subtype);
         }
    
         /**
          * Public constant media type for {@code application/underscore}.
          * @see #APPLICATION_UNDERSCORE_UTF8
          */
         public final static MediaType APPLICATION_UNDERSCORE;
    
         /**
          * Public constant media type for {@code application/underscore;charset=UTF-8}.
          */
         public final static MediaType APPLICATION_UNDERSCORE_UTF8;
    
         /**
          * A String equivalent of {@link CustomMediaType#APPLICATION_UNDERSCORE}.
          * @see #APPLICATION_UNDERSCORE_UTF8_VALUE
          */
         public final static String APPLICATION_UNDERSCORE_VALUE = "application/underscore";
    
         /**
          * A String equivalent of {@link CustomMediaType#APPLICATION_UNDERSCORE_UTF8}.
          * @see #APPLICATION_UNDERSCORE_VALUE
          */
         public final static String APPLICATION_UNDERSCORE_UTF8_VALUE = "application/underscore;charset=UTF-8";
    
         static {
             APPLICATION_UNDERSCORE = valueOf(APPLICATION_UNDERSCORE_VALUE);
             APPLICATION_UNDERSCORE_UTF8 = valueOf(APPLICATION_UNDERSCORE_UTF8_VALUE);
         }
     } 
```
自定义消息转换工具：

```java
    public class UnderScoreConverter extends AbstractJackson2HttpMessageConverter {
    
    
         public UnderScoreConverter(ObjectMapper objectMapper) {
             // 初始化消息转换器，该转换器支持自定义的媒体类型
             super(objectMapper, CustomMediaType.APPLICATION_UNDERSCORE, CustomMediaType.APPLICATION_UNDERSCORE_UTF8);
    
             // 自定义解析方式
             objectMapper.setSerializerFactory(objectMapper.getSerializerFactory()
                     .withSerializerModifier(new MyBeanSerializerModifier()));
             // 自定义日期解析方式（日期建议还是使用时间戳，再由调用方控制展示样式）
             objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")).setTimeZone(TimeZone.getTimeZone("GMT+8"));
             // 属性命名策略选用下划线方式
             objectMapper.setPropertyNamingStrategy(PropertyNamingStrategy.SNAKE_CASE);
             // 允许包含控制字符串结束等特殊字符
             objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);
         }
    
         class MyBeanSerializerModifier extends BeanSerializerModifier {
    
             @Override
             public List<BeanPropertyWriter> changeProperties(SerializationConfig config,
                                                              BeanDescription beanDesc,
                                                              List<BeanPropertyWriter> beanProperties) {
                 // 循环所有的beanPropertyWriter
                 for (int i = 0; i < beanProperties.size(); i++) {
                     BeanPropertyWriter writer = beanProperties.get(i);
                     // 判断字段的类型，如果是数组或集合则注册nullSerializer
    
                     // 注册null转空数组
                     if (isArrayType(writer)) {
                         writer.assignNullSerializer(new NullArrayJsonSerializer());
                     } else if (isMap(writer)) {
                         writer.assignNullSerializer(new NullObjectJsonSerializer());
                     } else if (isObjectType(writer)) {
                         writer.assignNullSerializer(new NullObjectJsonSerializer());
                     }
                 }
                 return beanProperties;
             }
    
             /**
              * 是否是数组
              */
             private boolean isArrayType(BeanPropertyWriter writer) {
                 Class<?> clazz = writer.getType().getRawClass();
                 return clazz.isArray() || Collection.class.isAssignableFrom(clazz);
             }
    
             /**
              * 是否是map
              *
              * @param writer 属性编辑器
              * @return
              */
             private boolean isMap(BeanPropertyWriter writer) {
                 Class<?> clazz = writer.getType().getRawClass();
                 return Map.class.isAssignableFrom(clazz);
             }
    
             /**
              * 是否是String
              */
             private boolean isStringType(BeanPropertyWriter writer) {
                 Class<?> clazz = writer.getType().getRawClass();
                 return CharSequence.class.isAssignableFrom(clazz) || Character.class.isAssignableFrom(clazz);
             }
    
             /**
              * 是否是数值类型
              */
             private boolean isNumberType(BeanPropertyWriter writer) {
                 Class<?> clazz = writer.getType().getRawClass();
                 return Number.class.isAssignableFrom(clazz);
             }
    
             /**
              * 是否是boolean
              */
             private boolean isBooleanType(BeanPropertyWriter writer) {
                 Class<?> clazz = writer.getType().getRawClass();
                 return clazz.equals(Boolean.class);
             }
    
             /**
              * 是否是对象
              *
              * @param writer 属性编辑器
              * @return
              */
             private boolean isObjectType(BeanPropertyWriter writer) {
                 Class<?> clazz = writer.getType().getRawClass();
                 if (isBooleanType(writer) || isNumberType(writer) || isStringType(writer) || clazz.isArray() || isMap(writer)) {
                     return false;
                 } else {
                     return true;
                 }
             }
         }
    
         /**
          * 处理数组集合类型的null值
          */
         public static class NullArrayJsonSerializer extends JsonSerializer<Object> {
             @Override
             public void serialize(Object value, JsonGenerator jsonGenerator,
                                   SerializerProvider serializerProvider) throws IOException {
                 jsonGenerator.writeStartArray();
                 jsonGenerator.writeEndArray();
             }
         }
    
         /**
          * 处理实体对象类型的null值
          */
         public static class NullObjectJsonSerializer extends JsonSerializer<Object> {
             @Override
             public void serialize(Object value, JsonGenerator jsonGenerator,
                                   SerializerProvider serializerProvider) throws IOException {
                 jsonGenerator.writeStartObject();
                 jsonGenerator.writeEndObject();
             }
         }
     }
```


Spring MVC初始化时让其加载自定义转换器：

```java  
    @Configuration
    public class CustomWebMvcConfigurationSupport extends WebMvcConfigurationSupport {
    
        // 自定义扩展消息转换器
        @Override
        protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    
    
            super.extendMessageConverters(converters);
            // 自定义仅是对默认转换器做个补充
            if (converters != null) {
                // 初始化默认转换器并加入到默认转换器集合中
                UnderScoreConverter underLineConverter = new UnderScoreConverter(
                        Jackson2ObjectMapperBuilder.json().applicationContext(getApplicationContext()).build());
                converters.add(underLineConverter);
            }
        }
    
        // 由于自定义WebMvcConfigurationSupport导致swagger初始化配置被覆盖，以下用于恢复swagger配置。
        @Override
        protected void addResourceHandlers(ResourceHandlerRegistry registry) {
            registry.addResourceHandler("swagger-ui.html")
                    .addResourceLocations("classpath:/META-INF/resources/");
            registry.addResourceHandler("/webjars/**")
                    .addResourceLocations("classpath:/META-INF/resources/webjars/");
        }
    }
```
以上配置启动项目后，调用方在请求头(Headers)中增加Accept:Application/underscore，返回值的转换器就能指定为自定义的转换器了。

