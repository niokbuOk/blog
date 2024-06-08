
### 1.通过PageHelper

PageHelper是MyBatis的一个分页插件。

**工作原理**：

1.拦截Mybatis的查询请求：PageHelper通过Mybatis的插件机制，拦截Executor(执行器)的查询方法。在执行查询之前，它会根据分页参数进行处理。

2.**设置分页参数**：在调用 Mapper 方法之前，使用 `PageHelper.startPage(pageNum, pageSize)` 设置分页参数。这个方法会将分页参数存储在一个 ThreadLocal 变量中，以便在之后的拦截器中使用。

3.**修改 SQL 查询**：在拦截到查询请求时，PageHelper 插件会根据存储的分页参数，修改原始的 SQL 查询语句。具体来说，它会在原始查询语句后面添加适当的 LIMIT 子句，以实现分页查询。

4.**执行分页查询**：修改后的 SQL 语句会被发送到数据库执行。数据库返回指定页的数据。

5.**获取总记录数**：除了分页查询外，PageHelper 还会执行一个 COUNT 查询，以获取总记录数。这个查询是自动生成的，用于计算分页信息，如总页数和总记录数。

6.**封装分页结果**：分页查询结果和总记录数会被封装到一个 `PageInfo` 对象中，返回给调用者。

### 详细步骤

以下是 PageHelper 分页的详细步骤：

1. **调用 `PageHelper.startPage`**：在执行查询方法之前，调用 `PageHelper.startPage(pageNum, pageSize)` 设置分页参数。

   ```
   java复制代码PageHelper.startPage(1, 10);
   List<User> users = userMapper.getAllUsers();
   PageInfo<User> pageInfo = new PageInfo<>(users);
   ```

   `startPage` 方法会将分页参数（页码和每页记录数）存储到 ThreadLocal 变量中。

2. **拦截 MyBatis 查询**：PageHelper 实现了 MyBatis 的拦截器接口（Interceptor），拦截 Executor 的查询方法。这个拦截器会在查询方法执行前被调用。

   ```
   java复制代码@Override
   public Object intercept(Invocation invocation) throws Throwable {
       // 获取当前的分页参数
       Page<?> page = getLocalPage();
       if (page != null) {
           // 修改 SQL 查询语句
           // 添加 LIMIT 子句
           MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
           BoundSql boundSql = ms.getBoundSql(invocation.getArgs()[1]);
           String originalSql = boundSql.getSql();
           String limitSql = originalSql + " LIMIT " + page.getStartRow() + ", " + page.getPageSize();
           // 修改 BoundSql 中的 SQL 语句
           Field sqlField = BoundSql.class.getDeclaredField("sql");
           sqlField.setAccessible(true);
           sqlField.set(boundSql, limitSql);
       }
       // 执行查询
       return invocation.proceed();
   }
   ```

3. **生成分页查询 SQL**：拦截器会在原始 SQL 语句后面添加 LIMIT 子句。例如，原始的 SQL 是：

   ```
   sql
   复制代码
   SELECT * FROM users
   ```

   修改后的 SQL 是：

   ```
   sql
   复制代码
   SELECT * FROM users LIMIT 0, 10
   ```

   这里的 `0` 是起始行号，`10` 是每页的记录数。

4. **执行分页查询**：MyBatis 执行修改后的 SQL 查询，返回当前页的数据。

5. **执行总记录数查询**：PageHelper 还会生成并执行一个 COUNT 查询，以获取总记录数。例如：

   ```
   sql
   复制代码
   SELECT COUNT(*) FROM users
   ```

6. **封装分页结果**：PageHelper 将分页查询结果和总记录数封装到 `PageInfo` 对象中，并返回给调用者。

   ```
   java复制代码public PageInfo(List<T> list) {
       this.list = list;
       this.total = list instanceof Page ? ((Page) list).getTotal() : 0;
       this.pages = list instanceof Page ? ((Page) list).getPages() : 1;
       // 其他分页信息的初始化
   }
   ```

## 2.通过RowBounds

`RowBounds` 是 MyBatis 提供的一种用于分页的方式。与 PageHelper 不同，`RowBounds` 通过设置偏移量和限制行数来实现分页。虽然 `RowBounds` 不如 PageHelper 功能强大，但它是 MyBatis 原生支持的分页方式，适用于简单的分页需求。

下面是如何使用 `RowBounds` 实现分页查询的详细步骤：

### 1. 设置 MyBatis 配置

确保你的项目已经配置好 MyBatis 环境。

### 2. 定义实体类和 Mapper 接口

假设我们有一个 `User` 实体类和一个 `UserMapper` 接口，用于查询用户信息。

```
java复制代码public class User {
    private Integer id;
    private String name;
    private String email;
    // getters and setters
}
java复制代码import org.apache.ibatis.annotations.Select;
import java.util.List;

public interface UserMapper {
    @Select("SELECT * FROM users")
    List<User> getAllUsers(RowBounds rowBounds);
}
```

### 3. 使用 `RowBounds` 进行分页查询

在 Service 层使用 `RowBounds` 进行分页查询。

```
java复制代码import org.apache.ibatis.session.RowBounds;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public List<User> getUsers(int pageNum, int pageSize) {
        // 计算偏移量
        int offset = (pageNum - 1) * pageSize;
        // 创建 RowBounds 对象
        RowBounds rowBounds = new RowBounds(offset, pageSize);
        // 执行分页查询
        return userMapper.getAllUsers(rowBounds);
    }
}
```

### 4. 在 Controller 层调用分页查询方法

在 Controller 层调用 Service 方法，获取分页结果并返回。

```
java复制代码import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;

@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/users")
    public List<User> getUsers(
            @RequestParam(defaultValue = "1") int pageNum,
            @RequestParam(defaultValue = "10") int pageSize) {
        return userService.getUsers(pageNum, pageSize);
    }
```

这是一种逻辑分页，先查询后，在内存中再进行分页。

## 3.MybatisPlus 分页插件

MyBatis-Plus 是一个 MyBatis 的增强工具，其提供了简洁而强大的分页功能。使用 MyBatis-Plus 实现分页查询非常简单，不需要手动编写复杂的 SQL 语句。下面是如何基于 MyBatis-Plus 实现分页功能的详细步骤：

### 1. 引入依赖

首先，在你的 Maven 或 Gradle 项目中引入 MyBatis-Plus 相关依赖。

#### Maven

```
xml复制代码<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.1</version>
</dependency>
```

#### Gradle

```
groovy
复制代码
implementation 'com.baomidou:mybatis-plus-boot-starter:3.4.1'
```

### 2. 配置 MyBatis-Plus

配置 MyBatis-Plus 使其支持分页功能。通常，这些配置会在 Spring Boot 的配置文件中进行。

#### application.yml

```
yaml复制代码mybatis-plus:
  mapper-locations: classpath:/mapper/*.xml
  type-aliases-package: com.example.domain
  global-config:
    db-config:
      id-type: auto
      logic-delete-field: deleted
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

### 3. 配置分页插件

在 MyBatis-Plus 中使用分页插件，只需简单配置即可。你可以在配置类中配置分页插件。

```
java复制代码import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return interceptor;
    }
}
```

### 4. 创建实体类和 Mapper 接口

#### 实体类

假设我们有一个 `User` 实体类：

```
java复制代码import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import lombok.Data;

@Data
@TableName("user")
public class User {
    @TableId
    private Long id;
    private String name;
    private Integer age;
}
```

#### Mapper 接口

为 `User` 实体类创建一个 Mapper 接口：

```
java复制代码import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface UserMapper extends BaseMapper<User> {
}
```

### 5. 实现分页查询

在 Service 层实现分页查询功能。

```
java复制代码import com.baomidou.mybatisplus.core.metadata.IPage;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public IPage<User> getUsersPage(int pageNum, int pageSize) {
        Page<User> page = new Page<>(pageNum, pageSize);
        return userMapper.selectPage(page, null);
    }
}
```

### 6. 在 Controller 层调用分页查询方法

在 Controller 层调用 Service 方法，获取分页结果并返回。

```
java复制代码import com.baomidou.mybatisplus.core.metadata.IPage;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/users")
    public IPage<User> getUsersPage(
            @RequestParam(defaultValue = "1") int pageNum,
            @RequestParam(defaultValue = "10") int pageSize) {
        return userService.getUsersPage(pageNum, pageSize);
    }
}
```

## 4.只用在SQL语句中添加limit

PageHelper和MyBtis-Plus 是物理分页，而RowBounds是逻辑分页。

