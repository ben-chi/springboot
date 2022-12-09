# 小杯的Springboot





## 自定义Converter

```java
@Bean
@Override
public FormattingConversionService mvcConversionService() {
   Format format = this.mvcProperties.getFormat();
   WebConversionService conversionService = new WebConversionService(new DateTimeFormatters()
         .dateFormat(format.getDate()).timeFormat(format.getTime()).dateTimeFormat(format.getDateTime()));
   addFormatters(conversionService); //开了个口子，留给自定义Converter
   return conversionService;
}

//调用this.configurers的addFormatters方法
@Override
protected void addFormatters(FormatterRegistry registry) {
	this.configurers.addFormatters(registry);
}

//addFormatters方法
@Override
public void addFormatters(FormatterRegistry registry) {
	for (WebMvcConfigurer delegate : this.delegates) {
		delegate.addFormatters(registry);
	}
}


private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

//获取容器中已经被定义的WebMvcConfigurer
@Autowired(required = false)
public void setConfigurers(List<WebMvcConfigurer> configurers) {
	if (!CollectionUtils.isEmpty(configurers)) {
		this.configurers.addWebMvcConfigurers(configurers);
	}
}

//上述addWebMvcConfigurers方法
public void addWebMvcConfigurers(List<WebMvcConfigurer> configurers) {
	if (!CollectionUtils.isEmpty(configurers)) {
		this.delegates.addAll(configurers);
	}
}

//因此我们需要的就是自定义WebMvcConfigurer类实现addFormatters方法，并放入容器中。在Springboot的上述自动化过程执行后，容器中的conversionService就会带有我们定义的Formatters。
```

## View及其ViewResolver

```java
//ViewResolver通过解析视图名获取View
public interface ViewResolver {
   View resolveViewName(String viewName, Locale locale) throws Exception;
}

//View通过render渲染结果，并赋给response
public interface View {
	void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
			throws Exception;

}
```

## MessageConverter

```java
//读request，写response时使用，例如json、xml格式的String与内存对象之间的转换
public interface HttpMessageConverter<T> {

   boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);

   boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);

   T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
         throws IOException, HttpMessageNotReadableException;

   void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
         throws IOException, HttpMessageNotWritableException;

}
```

### 自定义MessageConverter

```java
//所有的WebMvcConfigurer都被集成到了DelegatingWebMvcConfiguration中

//这里定义了被容器感知的所有MessageConverter
//先调用configureMessageConverters
//如果为空则默认构造addDefaultHttpMessageConverters
//最后使用extendMessageConverters扩展
protected final List<HttpMessageConverter<?>> getMessageConverters() {
	if (this.messageConverters == null) {
		this.messageConverters = new ArrayList<>();
		configureMessageConverters(this.messageConverters);
		if (this.messageConverters.isEmpty()) {
			addDefaultHttpMessageConverters(this.messageConverters);
		}
		extendMessageConverters(this.messageConverters);
	}
	return this.messageConverters;
}

//通过extendMessageConverters来增加MessageConverter
@Override
protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
	this.configurers.extendMessageConverters(converters);
}

//此处即为this.configurers.extendMessageConverters
//delegate就包含我们定义的WebMvcConfigurer
@Override
public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
	for (WebMvcConfigurer delegate : this.delegates) {
		delegate.extendMessageConverters(converters);
	}
}

//自动配置类WebMvcAutoConfiguration中配置的WebMvcAutoConfigurationAdapter包含于EnableWebMvcConfiguration中
//从而在EnableWebMvcConfiguration中配置的RequestMappingHandlerAdapter等等都会受到WebMvcAutoConfigurationAdapter和其余WebMvcConfigurer的影响

//因此我们要做的就是在自定义的WebMvcConfigurer中重写extendMessageConverters方法！！！
```

## 自定义ContentNegotiation

```java
//容器中的ContentNegotiationManager
public ContentNegotiationManager mvcContentNegotiationManager() {
   ContentNegotiationManager manager = super.mvcContentNegotiationManager();
   List<ContentNegotiationStrategy> strategies = manager.getStrategies();
   ListIterator<ContentNegotiationStrategy> iterator = strategies.listIterator();
   while (iterator.hasNext()) {
      ContentNegotiationStrategy strategy = iterator.next();
      if (strategy instanceof PathExtensionContentNegotiationStrategy) {
         iterator.set(new OptionalPathExtensionContentNegotiationStrategy(strategy));
      }
   }
   return manager;
}

//上述super.mvcContentNegotiationManager，可见ContentNegotiationManager的创建是通过初始化一个ContentNegotiationManager专用的configurer，再调用容器中所用的WebMvcConfigurer的configureContentNegotiation对该configurer进行配置。最后借助于工厂模式产出一个contentNegotiationManager
public ContentNegotiationManager mvcContentNegotiationManager() {
	if (this.contentNegotiationManager == null) {
		ContentNegotiationConfigurer configurer = new ContentNegotiationConfigurer(this.servletContext);
		configurer.mediaTypes(getDefaultMediaTypes());
		configureContentNegotiation(configurer);
		this.contentNegotiationManager = configurer.buildContentNegotiationManager();
	}
	return this.contentNegotiationManager;
}
//上面的Converter和MessageConverter的生产过程是Springboot生成默认List<Converter>和List<MessageConverter>，然后通过自定义WebMvcConfigurer加载额外的Converter和MessageConverter。
//ContentNegotiation的生产过程则是通过自定义WebMvcConfigurer修改配置类，最后通过修改完成的配置类生成最终的ContentNegotiation
//两者细节上有差别！！！
```

