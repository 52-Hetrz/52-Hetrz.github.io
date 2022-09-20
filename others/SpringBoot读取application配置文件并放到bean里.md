# SpringBoot读取application配置文件并放到bean里

使用@data注解是为了添加get和setter函数，@Component注解是为了返回当前类的Bean

```java
import lombok.Data;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
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

}
```



```properties
hibernate.show_sql=true
hibernate.format_sql=true
hibernate.hbm2ddl.auto=update
hibernate.packageScan=app.demo.bean
hibernate.current_session_context_class=org.springframework.orm.hibernate5.SpringSessionContext
```