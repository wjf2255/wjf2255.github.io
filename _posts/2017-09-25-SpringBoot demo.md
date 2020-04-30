---
title: SpringBoot 例子
tags: [SpringBoot]
layout: post
author: wjf
---


# 目录

-   [Spring boot](#orgf08c3f5)
    -   [利用 Gradle 配置 Spring Boot 项目](#org02fad72)
    -   [配置 jooq](#org09eb9e3)
    -   [配置 Web 项目](#org6bc141a)
    -   [配置 Spring-Security](#orgf78153b)
    -   [使用 JOOQ 访问数据库](#orgcdf107d)



<a id="orgf08c3f5"></a>

# Spring boot

spring boot 是为了响应 spring 框架依赖太多的 XML 配置和复杂的依赖管理而开发的框架。Spring Boot 让我们创建一个基于 Spring 的，独立的，产品级别的项目变得非常简单。Spring boot 将 Spring 平台和第三方库预先做了绑定，所以在使用 Spring Boot 过程中需要关注的问题非常少，配置量非常简单。

我们可以利用 Spring Boot 创建 Java 应用，并利用 java -jar 命令启动 Spring Boot 应用，也可以打包成传统的 war 方式部署。


<a id="org02fad72"></a>

## 利用 Gradle 配置 Spring Boot 项目

Gradle 是一个基于 JVM 的构建工具，和 Maven 功能类似，并且可以支持 Maven、Ivy 仓库，支持传递性依赖管理。Gradle 基于 Groovy,build 脚本配置可以直接使用 Groovy 编写，功能非常强大。

在项目根据路创建 build.gradle 文件，并引入 Spring Boot 插件：

    
    buildscript {
     ext {
         springBootVersion = '1.5.2.RELEASE'
     }
     repositories {
         mavenCentral()
     }
     dependencies {
         classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
     }
    }

如果需要打成 war 包，引入“war”插件即可：apply plugin:'war'。以下是完整的 Spring Boot 项目配置：

    
    buildscript {
     ext {
         springBootVersion = '1.5.2.RELEASE'
     }
     repositories {
         mavenCentral()
     }
     dependencies {
         classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
     }
    }
    
    apply plugin: 'java'
    apply plugin: 'org.springframework.boot'
    //apply plugin: 'war'             //如果需要打包成 war 包，配置上即可
    
    group 'jackframe'                 //项目组
    version '1.0-SNAPSHOT'            //版本号
    sourceCompatibility = 1.8
    
    repositories {
    mavenCentral()
    maven { url "http://repo.spring.io/libs-snapshot" }
    }
    
    dependencies {
    compile("org.springframework.boot:spring-boot-starter-web")
    compile("org.springframework.boot:spring-boot-starter-tomcat")
    //compile('org.springframework.boot:spring-boot-starter-security')
    compile("org.springframework.boot:spring-boot-starter-jooq")
    compile("org.springframework.boot:spring-boot-starter-aop")
    compile("com.alibaba:fastjson:1.2.37")
    compile("com.alibaba:druid:1.1.2")
    compile('mysql:mysql-connector-java')
    compile("joda-time:joda-time:2.9.9")
    compile("org.apache.commons:commons-lang3:3.6")
    //providedRuntime("org.springframework.boot:spring-boot-starter-tomcat")
    testCompile('org.springframework.boot:spring-boot-starter-test')
    }
    
    project.description = '''
    Spring Boot example
    '''

Gradle 的配置就结束了，但是要启动 Spring Boot 项目，还需要给 Spring Boot 一个程序入口：

    
    @SpringBootApplication
    @ServletComponentScan // 扫描使用注解方式的 servlet
    public class Application {
    
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);     //可以通过 args 来自定义一些程序参数。
        }
    }


<a id="org09eb9e3"></a>

## 配置 jooq

JOOQ 的使用请参见 JOOQ[官网](<https://www.jooq.org/>)。Jooq 是一个非常不错的 ORM 框架，它能够自动生成 Java 代码，并利用生成好的 Java 代码流式编写安全的 SQL 代码。为了更好自动生成 jooq 代码，我们往项目中添加一个 jooq.build 文件，内容如下：

    
       // Configure the Java plugin and the dependencies
    // ----------------------------------------------
    apply plugin: 'java'
    
    repositories {
        mavenLocal()
        mavenCentral()
    }
    
    dependencies {
        compile 'org.jooq:jooq:3.9.5'
    
        runtime 'mysql:mysql-connector-java:5.1.6'
        testCompile 'junit:junit:4.11'
    }
    
    buildscript {
        repositories {
            mavenLocal()
            mavenCentral()
        }
    
        dependencies {
            classpath 'org.jooq:jooq-codegen:3.9.5'
            classpath 'mysql:mysql-connector-java:5.1.6'
        }
    }
    
    
    // Use your favourite XML builder to construct the code generation configuration file
    // ----------------------------------------------------------------------------------
    def writer = new StringWriter()
    def xml = new groovy.xml.MarkupBuilder(writer)
            .configuration('xmlns': 'http://www.jooq.org/xsd/jooq-codegen-3.9.2.xsd') {
        jdbc() {    //数据库连接配置
            driver('com.mysql.jdbc.Driver') //数据库连接驱动
            url('jdbc:mysql://test.com:3306/crmdb?useUnicode=true&characterEncoding=utf8&autoReconnect=true&rewriteBatchedStatements=TRUE&zeroDateTimeBehavior=convertToNull') //数据库连接 URL
            user('daqi'test)   //数据库账号
            password('test')   //数据库密码
        }
        generator() {        //java 代码生成规则
            database() {     //数据库配置
                inputSchema("databasename")   //需要生成 Java 的数据库名称
                name("org.jooq.util.mysql.MySQLDatabase")  //指定数据库类型
                unsignedTypes(false)                       //不使用 unsigned 类型。（true 在 int(11) unsigined 会生成 jooq 的类型 UInteger）
            }
    
            // Watch out for this caveat when using MarkupBuilder with "reserved names"
            // - https://github.com/jOOQ/jOOQ/issues/4797
            // - http://stackoverflow.com/a/11389034/521799
            // - https://groups.google.com/forum/#!topic/jooq-user/wi4S9rRxk4A
            generate([:]) {
                pojos true     //生成表的值对象类
                daos true      //生成 简单的增删改查
            }
            target() {
                packageName('package')    //包名
                directory('src/main/java')
            }
        }
    }
    
    // Run the code generator
    // ----------------------
    org.jooq.util.GenerationTool.generate(
            javax.xml.bind.JAXB.unmarshal(new StringReader(writer.toString()), org.jooq.util.jaxb.Configuration.class)
    )

在项目中加入这个 jooq 配置文件，可以利用 gradle build 命令生成对应的 Java 代码。


<a id="org6bc141a"></a>

## 配置 Web 项目

我们一直秉承无 XML 的配置，我们继续朝这个方向努力。

org.springframework.context.annotation.Configuration 注解为我们脱离 XML 配置文件的苦海提供了非常便捷的途径。Configuration 注解的说明：“Indicates that a class declares one or more {@link Bean @Bean} methods and may be processed by the Spring container to generate bean definitions and service requests for those beans at runtime”，大意是说这个注解申明这个类的一个或者多个方法定义的类需要 Spring 容器托管。之前需要配置在 XML 中的 Bean，现在可以申明在配置类中。

由于在使用 Spring MVC 框架，为了方便我们定义控制层和视图，我们的配置类可以直接继承 WebMvcConfigurerAdapter。Spring MVC 框架提供 WebMvcConfigurer 接口，方便用户自定义 Spring MVC 的功能。基本上我们不需要自定义太多功能，为此 Spring MVC 非常周到的提供了一个 stub implementation 对象 WebMvcConfigurerAdapter。

    
    @Configuration
    public class WJFCRMConfig extends WebMvcConfigurerAdapter {

我们可以重载方法，自定义 Controller 和静态资源文件：

    
    @Override
     public void addViewControllers(ViewControllerRegistry registry) {
         registry.addViewController( "/" ).setViewName("forward:/public/index.html");
         registry.addViewController("/error").setViewName("forward:/public/error/error.html");
         registry.setOrder(Ordered.HIGHEST_PRECEDENCE );
     }
    
     @Override
     public void addResourceHandlers(ResourceHandlerRegistry registry) {
         registry.addResourceHandler("/static/**").addResourceLocations("classpath:/static/");
         registry.addResourceHandler("/public/**").addResourceLocations("classpath:/public/");
         registry.addResourceHandler("/css/**").addResourceLocations("classpath:/css/");
         registry.addResourceHandler("/js/**").addResourceLocations("classpath:/js/");
         registry.addResourceHandler("/favicon.ico").addResourceLocations("/").setCachePeriod(0);  //配置 favicon 图标
         super.addResourceHandlers(registry);
     }

我们在配置数据源时，需要读取很多配置信息，@Value 注解可以方便我们读取配置信息。

    
    @Value("${spring.datasource.url}")
    private String dbUrl;
    
    @Value("${spring.datasource.username}")
    private String username;
    
    @Value("${spring.datasource.password}")
    private String password;
    
    ...

@Value 还可以用在方法参数或者构造函数上，但是在这里我们只是利用它帮我们读取配置文件。

接下来我们开始配置我们的数据源：

    
    @Bean(destroyMethod = "close")
     @Primary
     public DataSource dataSourceA() {
         DruidDataSource datasource = new DruidDataSource();
    
         datasource.setUrl(dbUrl);
         datasource.setUsername(username);
         datasource.setPassword(password);
         datasource.setDriverClassName(driverClassName);
    
         // configuration
         datasource.setInitialSize(initialSize);
         datasource.setMinIdle(minIdle);
         datasource.setMaxActive(maxActive);
         datasource.setMaxWait(maxWait);
         datasource.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
         datasource.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
         datasource.setValidationQuery(validationQuery);
         datasource.setTestWhileIdle(testWhileIdle);
         datasource.setTestOnBorrow(testOnBorrow);
         datasource.setTestOnReturn(testOnReturn);
         datasource.setPoolPreparedStatements(poolPreparedStatements);
         datasource.setMaxPoolPreparedStatementPerConnectionSize(maxPoolPreparedStatementPerConnectionSize);
         try {
             datasource.setFilters(filters);
         } catch (SQLException e) {
    
         }
         datasource.setConnectionProperties(connectionProperties);
    
         return datasource;
     }
    
     @Bean
     public DataSourceTransactionManager transactionManagerA() {
         DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
         transactionManager.setDataSource(dataSourceA());
         return transactionManager;
     }
    
     @Bean
     public DataSourceConnectionProvider dataSourceConnectionProviderA(
             @Qualifier("dataSourceA") DataSource dataSource) {
         return new DataSourceConnectionProvider(
                 new TransactionAwareDataSourceProxy(dataSource));
     }
    
     @Bean
     public SpringTransactionProvider transactionProviderA(
             @Qualifier("transactionManagerA") DataSourceTransactionManager txManagerW) {
         return new SpringTransactionProvider(txManagerW);
     }
    
     @Bean
     public DefaultConfiguration configurationA(
             @Qualifier("dataSourceConnectionProviderA") DataSourceConnectionProvider dataSourceConnectionProviderA,
             @Qualifier("transactionProviderA") SpringTransactionProvider transactionProviderA
     ) {
         DefaultConfiguration configuration = new DefaultConfiguration();
         configuration.setSQLDialect(SQLDialect.MYSQL);
         configuration.set(dataSourceConnectionProviderA);
         configuration.set(transactionProviderA);
         if (this.recordMapperProvider != null) {
             configuration.set(this.recordMapperProvider);
         }
         if (this.settings != null) {
             configuration.set(this.settings);
         }
         configuration.set(this.recordListenerProviders);
         configuration.set(this.visitListenerProviders);
         configuration.set(new DefaultExecuteListenerProvider(
                 exceptionTranslator()
         ));
         return configuration;
     }
    
     @Bean
     public DefaultDSLContext dslContextA(@Qualifier("configurationA") DefaultConfiguration defaultConfiguration) {
         return new DefaultDSLContext(defaultConfiguration);
     }
    
     @Bean
     ExceptionTranslator exceptionTranslator() {
         return new ExceptionTranslator();
     }
    
     public class ExceptionTranslator extends DefaultExecuteListener {
         @Override
         public void exception(ExecuteContext ctx) {
             // [#4391] Translate only SQLExceptions
             if (ctx.sqlException() != null) {
                 SQLDialect dialect = ctx.dialect();
                 SQLExceptionTranslator translator = (dialect != null)
                         ? new SQLErrorCodeSQLExceptionTranslator(dialect.thirdParty().springDbName())
                         : new SQLStateSQLExceptionTranslator();
    
                 ctx.exception(translator.translate("jOOQ", ctx.sql(), ctx.sqlException()));
             }
         }
     }

我们可以通过定义不同的 DataSource 对象，定义多个数据源。在使用 Jooq 的时候，需要自定义一个 ExceptionTranslator。


<a id="orgf78153b"></a>

## 配置 Spring-Security

Spring Security 为 Java EE 项目提供了非常全面的的安全服务，提供了非常多的验证方式，我们开始体验一下 Spring Security。

添加 Spring Security 配置信息：

    
    @Configuration
    @EnableWebSecurity
    @EnableGlobalMethodSecurity(prePostEnabled=true) //让方法支持@PreAuthorize
    public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

WebSecurityConfigurer 接口和 Spring MVC 框架提供的 WebMvcConfigurer 接口功能类似，也是为了方便我们自定义 Spring Security 功能。并且 Spring Security 框架也提供了基于各种技术的实现，这里我们使用 WebSecurityConfigurerAdapter。

本次 Spring Security 配置主要分成静态资源过滤，权限校验配置，以及一些自定义的配置。


### 静态资源过滤

对于一些静态资源，比如图片、js 脚本等，我们不需要做权限校验，需要告诉 Spring Security 过滤这些资源。我们可以通过重载 configure(WebSecurity web)方法实现这一功能。

    
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/assets/**", "/static/**", "/public/**", "/js/**", "/images/**","/**/*.jsp","**/favicon.ico");
    }

对于任何符合以上通配符地址的，Spring Security 都不做任何校验，直接通过。


### 权限和自定义配置

Spring Security 非常主要的配置项。我们将会配置那些 URL 地址可以直接访问，那些 URL 地址需要过滤，并且可以自定义一些校验规则等。

    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
    
        http.sessionManagement().maximumSessions(2*60*60);
    
        http
                .authorizeRequests()
                    .antMatchers("/error", "/api/v1/captcha", "/api/v1/test", "/favicon.ico", "/")
                    .permitAll()
                    .anyRequest().authenticated()
                    .and()
                .httpBasic()
                    .authenticationEntryPoint(custmoAuthenticationEntryPoint())
                    .and()
                .logout()
                    .logoutUrl("/api/v1/logout")
                    .permitAll()
                    .logoutSuccessHandler(new CustomLogoutSuccessHandler())
                    .addLogoutHandler(logoutHandler())
                ;
        http.addFilter(customLoginFilter());
        http.csrf().disable();
    }

本人本地的 server.session.timeout=1800 没有生效，所以配置 http.sessionManagement().maximumSessions(2\*60\*60); 用于控制 session 的超时时间。

http.csrf().disable(); 用于取消 csrf 功能。该功能大致是在 get 表单时，给一个<sub>csrf</sub>=随机码，并在表单提交时，校验这个随机码是否正确。

其他的配置主要是告诉 Spring Security 完全开放的 URL 地址和登出的 URL 地址。其他 URL（除静态资源外）都是需要做权限校验的。并自定义了一些校验规则，比如登录和登出处理。


### 自定义处理

由于默认的登录方式只校验账号和密码，但是我本地处理账号密码外，还需要校验验证码，所以需要做自定义。

    
    @Bean
    public CustomLoginFilter customLoginFilter() throws Exception {
        AuthenticationManager authenticationManager = this.authenticationManager();
        CustomAuthenticationFailureHandler customAuthenticationFailureHandler = new CustomAuthenticationFailureHandler();
        CustomSimpleUrlAuthenticationSuccessHandler customSimpleUrlAuthenticationSuccessHandler = new CustomSimpleUrlAuthenticationSuccessHandler();
        CustomLoginFilter customLoginFilter = new CustomLoginFilter(authenticationManager, customAuthenticationFailureHandler, customSimpleUrlAuthenticationSuccessHandler);
        return customLoginFilter;
    }
    
    private class CustomAuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {
    
        @Override
        public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
    
            ResponseForm form;
            if (exception != null) {
                form = ResponseForm.exceptionResult(ExceptionConvertUtil.convertException(exception));
                response.setStatus(HttpStatus.UNAUTHORIZED.value());
            } else {
                form = ResponseForm.exceptionResult(UserLoginFailure);
                response.setStatus(HttpStatus.UNAUTHORIZED.value());
            }
            response.setContentType("application/json");
            response.setCharacterEncoding("utf-8");
            response.getOutputStream().write(JSON.toJSONString(form).getBytes("utf-8"));
        }
    }
    
    private class CustomSimpleUrlAuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {
    
    
    
        @Override
        protected void handle(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
    
            RequestDispatcher dispatcher = request.getRequestDispatcher("/api/v1/loginSuccess");
            dispatcher.forward(request, response);
        }
    }
    
    /**
     * UsernamePasswordAuthenticationFilter 的父类 AbstractAuthenticationProcessingFilter
     * 会使用自己的 AuthenticationFailureHandler AbstractAuthenticationProcessingFilter 如下：
     * private AuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
     *  private AuthenticationFailureHandler failureHandler = new SimpleUrlAuthenticationFailureHandler();
     *
     *  导致 http.formLogin().failureUrl("/api/v1/loginFailure") 不起作用。
     *
     *  所以如果使用自定义 UsernamePasswordAuthenticationFilter 同时又需要配置认证失败的跳转链接时，需要将自定义失败连接配置到自定义
     *  UsernamePasswordAuthenticationFilter 中
     *
     */
    private class CustomLoginFilter extends UsernamePasswordAuthenticationFilter {
    
        Logger logger = LoggerFactory.getLogger(this.getClass());
    
        public CustomLoginFilter(AuthenticationManager authenticationManager,
                                 AuthenticationFailureHandler authenticationFailureHandler,
                                 CustomSimpleUrlAuthenticationSuccessHandler customSavedRequestAwareAuthenticationSuccessHandler) {
            AntPathRequestMatcher requestMatcher = new AntPathRequestMatcher("/api/v1/login", "POST");
            this.setRequiresAuthenticationRequestMatcher(requestMatcher);
            this.setAuthenticationManager(authenticationManager);
            this.setAuthenticationFailureHandler(authenticationFailureHandler);
            this.setAuthenticationSuccessHandler(customSavedRequestAwareAuthenticationSuccessHandler);
    
        }
    
        @Override
        public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
            if (!request.getMethod().equals("POST")) {
                throw new AuthenticationServiceException(
                        "Authentication method not supported: " + request.getMethod());
            }
    
            String account;
            String password;
    
            if (SecurityContextHolder.getContext().getAuthentication() == null || SecurityContextHolder.getContext().getAuthentication()
                    .getPrincipal() == null) {
    
                Map<String, Object> paramMap = new HashMap<>();
                try {
                    paramMap = RequestUtil.parseRequestParam(request);
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
    
                String verification = (String)paramMap.get("verifyCode");
                String captcha = (String) request.getSession().getAttribute(Constants.CAPTCHA_SESSION_KEY);
    
                //每次认证都删除回话中的验证码
                request.getSession().removeAttribute(Constants.CAPTCHA_SESSION_KEY);
                if (captcha == null || !captcha.equals(verification.toLowerCase())) {
                    throw new CaptchaNotMatch("captcha code not matched!");
                }
    
                account = (String)paramMap.get("account");
                password = shaPasswordEncoder.encodePassword((String)paramMap.get("password"), Constants.PASSWORD_SALT);
    
                if (account == null) {
                    account = "";
                }
    
                if (password == null) {
                    password = "";
                }
            } else {
                Account account =  (Account) SecurityContextHolder.getContext().getAuthentication()
                        .getPrincipal();
                account = account.getUsername();
                password = account.getPassword();
            }
    
            account = account.trim();
    
            UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
                    account, password);
            // Allow subclasses to set the "details" property
            setDetails(request, authRequest);
    
            return this.getAuthenticationManager().authenticate(authRequest);
        }
    }

CustomLogoutSuccessHandler 和 LogoutHandler 只是为了将 302 跳转转成直接返回错误信息，这里就不贴代码了。

Spring Security 验证账号密码时，需要我们告诉 Spring Security 如何根据用户名称获取账号信息。我们需要实现 UserDetailsService 接口。

    
    @Service
    public class UserDetailsServiceImpl implements UserDetailsService {
    
        @Autowired
        AccountEntity accountEntity;
    
        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            UserDetails userDetails = accountEntity.findAccount(username);
            if (userDetails == null) {
                throw new UsernameNotFoundException("用户信息不存在！");
            }
            return userDetails;
        }
    }

配置这些后，我们就可以利用@PreAuthorize 做访问权限控制了。

    
    @RequestMapping("/api/v1/accounts")
    @PreAuthorize("hasAnyAuthority('ADMIN', 'CS_LEADER')")
    public @ResponseBody ResponseForm accounts(@RequestParam int pageNo, @RequestParam int pageSize,
                                               HttpServletRequest request) {
    
        ...
    }

@PreAuthorize("hasAnyAuthority('ADMIN', 'EMPLOYEE')")表示只有拥有“ADMIN”和“EMPLOYEE”的用户才能够访问。


<a id="orgcdf107d"></a>

## 使用 JOOQ 访问数据库

我们已经在配置文件中定义了好了“configurationA”和“dslContextA”，并且利用 jooq.gradle 文件构建数据库 ORM 对象。之后生成数据库表 userdb.user 对应的 UserDao 和 User pojo 类。

    
    @Autowired
     @Qualifier("configurationA")
     protected DefaultConfiguration configuration;
    
    UserDao userDao = new UserDao(configuration);

就可以对表 userdb.user 做增删改查操作了。

比如查询：

    
    List<User> userList = using(configuration).selectFrom(USER).where(USER.ID.eq(1)).fetch().map(mapper());

参考资料：

<https://docs.spring.io/spring-security/site/docs/5.0.0.M3/reference/htmlsingle/#getting-started>

<https://github.com/openforis/calc/blob/master/calc-core/src/main/java/org/openforis/calc/persistence/jooq/ExceptionTranslator.java>

