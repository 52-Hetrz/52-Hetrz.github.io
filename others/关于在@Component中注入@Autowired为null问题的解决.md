---

typora-copy-images-to: images
typora-root-url: images
---

## 关于在@Component中注入@Autowired为null问题的解决

### 问题：

```java
@Configuration
@Data
public class HibernateConfig {


    @Value("${hibernate.current_session_context_class}")
    private String currentSessionContext ;
    @Value("${hibernate.show_sql}")
    private boolean showSql;
    @Value("${hibernate.format_sql}")
    private boolean formatSql;
    @Value("${hibernate.hbm2ddl.auto}")
    private String hbm2ddlAuto ;
    @Value("${hibernate.packageScan}")
    private String packageScan;

    public HibernateConfig(){
        System.out.println("HibernateConfig" );
    }

}
```

```java
@Component
public class HibernateSessionFactoryBuilder {

    @Autowired
    private ResourceLoader rl;
    
    @Autowired
    HibernateConfig hibernateConfig;
    
    public HibernateSessionFactoryBuilder(){
        System.out.println("Session Builder");
    }
    
}
```

其中注入的HibernateConfig显示为null，原因是在@Component注解将当前HibernateSessionFactoryBuilder类加载到Spring容器当中时，@Autowired注解注入的HibernateConfig是作为一个属性一起加载到容器中的，此时注入的bean还没有完成自动装载，所以无法注入，因此显示为null。

为了测试，我在HibenateConfig和HibernateSessionFactoryBuilder两个类中均添加了一个构造方法，用来测试Spring在装载两个类的bean时的先后顺序，输出结果如下：

![image-20211129105910622](/images/image-20211129105910622.png)

这就证明先创建的SessionBuilder，后创建的HibernateConfig，所以在SessionBuilder中注入HibernateConfig时，HibernateConfig还未完成自动装载，因此注入为null。（为了排除自动装载的随机性，我测试了多次，发现Spring挂载这两个类的顺序是确定的。

### 解决办法：

```java
@Component
public class HibernateSessionFactoryBuilder {

    @Autowired
    private ResourceLoader rl;

    private static HibernateConfig hibernateConfig;

    @Autowired
    public void setHibernateConfig(HibernateConfig hibernateConfig){
        HibernateSessionFactoryBuilder.hibernateConfig = hibernateConfig;
    }
}
```

@Autowired放在方法上，会在类装载完成之后自动注入参数，然后执行一边方法。这样就可以解决了，注意参数要设置为static