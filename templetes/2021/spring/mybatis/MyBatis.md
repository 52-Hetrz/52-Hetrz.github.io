---
typora-copy-images-to: images
typora-root-url: images
---

## MyBatis

#### 1. 在pom.xml中添加相关依赖

```xml
<dependency>
   <groupId>org.mybatis.spring.boot</groupId>
   <artifactId>mybatis-spring-boot-starter</artifactId>
   <version>1.3.0</version>
</dependency>
```

#### 2. 在application.yml文件中设置MyBatis相关配置

```yaml
mybatis:
  #映射文件在项目中的位置
  mapper-locations: classpath:mapping/*Mapping.xml
  #对应数据库的dao文件在项目中的包位置
  type-aliases-package: com.example.mybatis.dao				
```

#### 3.相关文件创建

在相应位置创建映射文件以及与之对应的mapper和dao文件

![image-20211116121711356](/image-20211116121711356.png)

UserMapping.xml配置文件文件如下：

* mapper标签中的namespace属性指定了该配置文件对应的mapper文件
* resultMap标签中的type属性指定了改配置文件对应的dao类
* result标签建立数据库中的列和dao类中的属性的映射

```xml
<?xml version="1.0" encoding="UTF-8" ?>

<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.mybatis.mapper.UserMapper">
    <resultMap id="UserResultMap" type="com.example.mybatis.dao.UserDao">
        <result column="id" property="id" jdbcType="INTEGER"/>
        <result column="name" property="name" jdbcType="VARCHAR"/>
        <result column="password" property="password" jdbcType="VARCHAR"/>
        <result column="mail" property="mail" jdbcType="VARCHAR"/>
        <result column="image" property="image" jdbcType="VARCHAR"/>
    </resultMap>
    
    <insert id="insertUser">
        insert
        into user (name, password, mail, image)
        values(#{name}, #{password}, #{mail}, #{image})
    </insert>

    <delete id="deleteUser">
        delete
        from user
        where id = #{id}
    </delete>

    <select id="selectAllUsers"  resultMap="UserResultMap">
        select *
        from user
    </select>
    
    <select id="selectUserIdByName" resultType="INTEGER">
        select id
        from user
        where name = #{name}
    </select>

</mapper>
```

UserDao文件：

```java
public class UserDao {

    private Integer id;
    private String name;
    private String password;
    private String mail;
    private String image;
    
    public UserDao(){}
    
    public UserDao(String name, String password, String mail, String image){
        this.setName(name);
        this.setImage(image);
        this.setPassword(password);
        this.setMail(mail);
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getMail() {
        return mail;
    }

    public void setMail(String mail) {
        this.mail = mail;
    }

    public String getImage() {
        return image;
    }

    public void setImage(String image) {
        this.image = image;
    }
}
```

UserMapper文件：

```java
@Mapper
public interface UserMapper {
    void insertUser(UserDao user);
    Integer selectUserIdByName(String name);
    void deleteUser(int id);
    ArrayList<UserDao> selectAllUsers();
}
```

#### 4.测试

服务层Service：

![image-20211116153518810](/image-20211116153518810.png)

UserService:

```java
@Service
public interface UserService {
    Integer insertUser(String name, String password, String mail, String image);
    void deleteUser(Integer id);
    ArrayList<UserDao> selectAllUsers();
}
```

UserServiceImpl:

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    UserMapper userMapper;

    @Override
    public Integer insertUser(String name, String password, String mail, String image) {
        if(userMapper.selectUserIdByName(name) == null){
            userMapper.insertUser(new UserDao(name,password,mail,image));
            return userMapper.selectUserIdByName(name);
        }
        return null;
    }

    @Override
    public void deleteUser(Integer id) {
        userMapper.deleteUser(id);
    }

    @Override
    public ArrayList<UserDao> selectAllUsers() {
        return userMapper.selectAllUsers();
    }
}
```

Controller:

```java
@RestController
public class UserController {

    @Autowired
    UserServiceImpl userService;

    @RequestMapping(value = "/user/insert/{name}/{password}/{mail}/{image}",method = RequestMethod.POST)
    public MessageResult insertUser(@PathVariable("name")String name,
                                    @PathVariable("password")String password,
                                    @PathVariable("mail")String mail,
                                    @PathVariable("image")String image
                                    ){
        Integer id = userService.insertUser(name,password,mail,image);
        if(id!=null){
            return new MessageResult(id,200,"success");
        }else{
            return new MessageResult(null,400,"error");
        }
    }

    @RequestMapping(value = "/user/selectAll",method = RequestMethod.POST)
    public MessageResult getAllUsers(){
        ArrayList<UserDao> result = userService.selectAllUsers();
        if(result!=null){
            return new MessageResult(result,200,"success");
        }else{
            return new MessageResult(null, 400,"error");
        }
    }

}
```