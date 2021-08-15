# 权限系统初识-基于rbac模型设计一个简单的权限


### 1. 认识权限

#### 1. 什么是权限

权限在一个系统中占据着十分重要的地位，一些功能只有特定角色用户才能访问，要做权限判断。

#### 2. 权限的分类

1. 登录权限

这个是所有权限的基础，也是接触最多的权限了， 所以这里就不在描述了

2. 功能系权限

对每个用户的系统权限精准控制。不同的人拥有不同的权限。
粗粒度权限管理，对资源类型的权限管理。资源类型比如：菜单、url连接、用户添加页面、用户信息、类方法、页面中按钮。 如：超级管理员可以访问用户添加页面、用户信息等全部页面。 部门管理员可以访问用户信息页面包括 页面中所有按钮。

细粒度权限管理，对资源实例的权限管理。数据级别的权限管理。如：部门经理只可以访问本部门的员工基本信息但不能查看身份证，用户只可以看到自己的全部信息。

### 2. 权限的设计原理

#### 1. RBAC 权限模型

权限的设计模型有很多方式，本案例中使用的是 RBAC 权限模型。RBAC 权限模型就是基于角色的访问控制。其基本思想是，对系统操作的各种权限不是直接授予具体的用户，而是在用户集合与权限集合之间建立一个角色集合。每一种角色对应一组相应的权限。一旦用户被分配了适当的角色后，该用户就拥有此角色的所有操作权限。这样做的好处是，不必在每次创建用户时都进行分配权限的操作，只要分配用户相应的角色即可，而且角色的权限变更比用户的权限变更要少得多，这样将简化用户的权限管理，减少系统的开销。

- 模型设计：
用户---> 角色 ---> 权限 ---> 功能

而用户和角色 角色和权限 从双方的角度看都是多对多的关系，所以都需要添加中间表进行关联

- 模型设计：
  用户--->用户角色关联表--->角色--->角色权限关联表--->权限--->功能

  

- 优点：简化权限和用户之间的关系，易扩展和维护。

#### 2. domain之间的关系分析

用户与角色：站在用户的角度，一个用户可能存在多个角色。站在角色的角度，一个角色有多个用户。用户与角色多对多的关系。

角色与权限：多对多的关系。

### 3. 项目解析

这里对项目中的一些思路和一些知识点进行一个总结，而关于前端的一些内容这里就不做描述了

#### 1. Mybatis的分页插件 - PageHelper

PageHelper是Mybatis集成的一款分页插件。支持常见的RowBounds(PageRowBounds)，
PageHelper.startPage 方法调用，Mapper 接口参数调用等等，使用起来十分方便。

1. 在 maven 中导入依赖

``` xml
<dependency>
  <groupId>com.github.pagehelper</groupId>
  <artifactId>pagehelper-spring-boot-starter</artifactId>
  <version>1.2.5</version>
</dependency>
```

2. 使用 PageHelper.startPage() 来进行分页

PageHelper.startPage(startPage, limitSize) 这个方法在调用之后会自动的将紧跟着 startPage 之后的第一个 select 语句进行分页，同时会将集合封装成 Page 对象。同时在 Page 对象中有查询条数、查询结果等等一系列的结果。

``` java
//要进行分页
PageHelper.startPage(roleQuery.getCurrentPage(),roleQuery.getPageSize());
//查询数据
List<Role> roles = roleMapper.selectAllByRoleName(roleQuery.getRoleName());
```

3. PageInfo结果封装。

PageInfo 类是对 Page 结果的进一步封装。里面提供了很多实用的属性例如是否是最后一页，是否是第一页等等。。
要使用 PageInfo 作为返回结果只需要在 进行一次封装就可以。

``` java
//把集合封装到PageInfo里面
PageInfo<Role> pageInfo = new PageInfo<>(roles);
```

4. PageHelper的函数式接口，ISelect 接口方式

- doSelectPage()

``` java
Page<Country> page = PageHelper.startPage(1, 10).doSelectPage(new ISelect() {
    @Override
    public void doSelect() {
        countryMapper.selectGroupBy();
    }
});
//jdk8 lambda用法
Page<Country> page = PageHelper.startPage(1, 10).doSelectPage(()-> countryMapper.selectGroupBy());
```

- doSelectPageInfo()

``` java
//也可以直接返回PageInfo，注意doSelectPageInfo方法和doSelectPage
pageInfo = PageHelper.startPage(1, 10).doSelectPageInfo(new ISelect() {
    @Override
    public void doSelect() {
        countryMapper.selectGroupBy();
    }
});
//对应的lambda用法
pageInfo = PageHelper.startPage(1, 10).doSelectPageInfo(() -> countryMapper.selectGroupBy());
```

#### 2. 对角色进行授权

角色授权页面的一个主要的点就是对权限的父权限和子权限进行封装。这里是要进行自关联查询，
要以父权限为主表。

``` sql
SELECT parent.*, child.id child_id, child.name child_name FROM permission parent
        LEFT JOIN permission child ON parent.id = child.parent_id
        WHERE parent.parent_id is null
```
而将结果封装到 domain 中去的话还需要进行一个 结果集的映射

``` java
public class Permission {
    private Long id;
    //权限名
    private String name;

    //权限url地址
    private String url;

    //权限对应的菜单
    private Menu menu;

    //子权限对应的父权限
    private Permission parent;

    //装子权限的
    private List<Permission> children = new ArrayList<>();
}
```

``` xml
<resultMap id="selectAllPermissionsType" type="Permission">
        <id column="id" property="id"/>
        <result property="name" column="name"/>
        <result property="url" column="url"/>
        
        <collection property="children" columnPrefix="child_" ofType="Permission">
            <id column="id" property="id"/>
            <result property="name" column="name"/>
        </collection>
</resultMap>
```

而回显的话，则是根据 角色id 直接查询 role_permission 关联表就可以了，如果要稳妥一点可以在 关联 permission表，从 permission 表中获取其权限id。

#### 3. 授权的保存

为了简化逻辑，可以在授权的时候直接将 当前角色的所有权限全部删除，在重新授权保存role_permission表中的数据。

#### 4. 动态菜单的一个实现

动态菜单的实现主要还是多表查询，根据用户id查出对应的角色信息，而通过角色信息查出权限，最后在根据权限就可以筛选出对应权限的菜单了。
当然这里的 sql 只是一个示例。
``` sql
SELECT m.*, mp.id mp_id, mp.name mp_name, mp.url mp_url FROM employee e
        JOIN employee_role er ON e.id = er.employee_id
        JOIN role_permission rp ON rp.role_id = er.role_id
        JOIN permission p ON rp.permission_id = p.id
        JOIN menu m ON p.menu_id = m.id
        JOIN menu mp ON m.parent_id = mp.id
        WHERE e.id = #{id}
```

将结果封装到 domain 中

``` java
public class Menu {

    /**主键id*/
    private Long id;
    /**菜单名*/
    private String name;
    /**菜单url*/
    private String url;
    /**菜单图标*/
    private String icon;
    /**父及菜单*/
    private Menu parentMenu;
    /**子菜单*/
    private List<Menu> children = new ArrayList<>();
}
```

``` xml
<resultMap id="menuType" type="Menu">
        <id column="id" property="id"/>
        <result property="name" column="name"/>
        <result property="url" column="url"/>
        <result property="icon" column="icon"/>
        <association property="parentMenu" javaType="Menu" columnPrefix="mp_">
            <id column="id" property="id"/>
            <result property="name" column="name"/>
            <result property="url" column="url"/>
        </association>
</resultMap>
```

而由于前台需要的数据格式是 父菜单-子菜单，所以还需要在进行一层封装

```java
//创建一个空集合(专门来装一级菜单的)
List<Menu> parents = new ArrayList<>();
List<Menu> menus = menuMapper.selectMenuByEmployeeId(employee.getId());
//定义一个map集合，该map集合是专门来装父级菜单的
Map<Long, Menu> map = new HashMap<>();
for (Menu menu : menus) {
  //父级菜单
  Menu parent = menu.getParent();
  //判断map   key中是否有父菜单对应的id,如果没有，则把父菜单添加到map集合中
  if(!map.containsKey(parent.getId())){
    map.put(parent.getId(), parent);
  }
  //拿到对应的父菜单
  Menu p =  map.get(parent.getId());
  //父菜单获取到子菜单集合之后，再进行添加子菜单
  p.getChildren().add(menu);
}
//拿到map中所有的v值
Collection<Menu> values = map.values();
//集合添加元素
parents.addAll(values);
return parents;
```

#### 5. 登录拦截器的配置

在 SpringBoot 中由于没有了 application.xml 配置文件，所以要注册拦截器就需要写一个配置类，同时去实现 WebMvcConfigurer 这个接口，重写其中的 `addInterceptors` 方法就可以了。

- 编写拦截器

``` java
//将拦截器交给 spring 管理
@Component
public class LoginInterceptor implements HandlerInterceptor {

    /**
     * 在执行目标方法之前，先执行此方法
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HttpSession session = request.getSession();
        //在session中获取登录用户
        Object loginUser = session.getAttribute("USER_IN_SESSION");
        if(loginUser == null){
            response.sendRedirect("/login");
            return false;//不放行
        }
        return true;//放行
    }
}
```

- 注册拦截器

添加一个类实现 WebMvcConfigurer 接口，在类上添加 `@Configuration` 注解，表示这就是SpringBoot的配置类。类似于之前的 application.xml 配置文件

``` java
@Configuration//配置类  类似于以前的xml文件
public class MyWebMvcConfigurer implements WebMvcConfigurer {

    @Autowired
    private LoginInterceptor loginInterceptor;

    @Autowired
    private PermissionInterceptor permissionInterceptor;

    /**
     * 注册拦截器
     * @param registry
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //添加登录拦截器
        registry.addInterceptor(loginInterceptor).addPathPatterns("/**")
                .excludePathPatterns("/login").excludePathPatterns("/assets/**");

        //添加权限拦截器
        registry.addInterceptor(permissionInterceptor).addPathPatterns("/**")
                .excludePathPatterns("/login").excludePathPatterns("/assets/**");
    }

    /**
     * 避免  单独写个controller去转发请求
     * @param registry
     */
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        //如果你请求的资源是/nopermission，我就跳转到nopermission.html的界面
        registry.addViewController("/nopermission").setViewName("nopermission");
    }
}
```

- 对权限进行拦截

如果当前页面在权限控制下，并且当前用户没有该权限则跳转到没有权限页面，否则放行

```
@Component
public class PermissionInterceptor implements HandlerInterceptor {

    @Autowired
    private PermissionMapper permissionMapper;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //获取当前请求的uri
        String uri = request.getRequestURI();
        //获取所有的权限(特点资源)
        List<String> urls = permissionMapper.selectAllPermissionUrl();
        //如果包含，则判断当前用户是否有权限访问uri的资源
        if(urls.contains(uri)){
            //获取当前登录人
            Employee employee = (Employee) request.getSession().getAttribute("USER_IN_SESSION");
            //获取当前用户拥有的权限
            List<String> permissions = permissionMapper.selectAllPermissionUrlByEmployeeId(employee.getId());
            if(permissions.contains(uri)){
                return true;
            }else{
                //跳转到没有权限的页面
                response.sendRedirect("/nopermission");
                return false;
            }
        }
        return true;
    }
}
```
