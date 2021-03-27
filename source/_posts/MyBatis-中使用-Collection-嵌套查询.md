---
title: MyBatis 中使用 Collection 嵌套查询
date: 2018-01-01 00:47:05
tags:
    - Java
    - MyBatis
categories: 
    - Java
    - MyBatis
---

> 当使用 MyBatis 进行查询的时候如果一个 JavaBean 中包含另一个 JavaBean 或者 Collection 时，可以通过 MyBatis 的嵌套查询来获取需要的结果;
> 以下以用户登录时的角色和菜单直接的关系为例使用嵌套查询

------------

## JavaBean 

- RoleModel
```
public class RoleModel {
    private Integer id;
    private String name;
    private Boolean isActive;
    private String description;
    private Date lastUpdateTime;
    private List<MenuModel> menus;
    ···
}   
```

- MenuModel
```
public class MenuModel {
    private Integer id;
    private String value;
    private String displayValue;
    private String url;
    private Integer category;
    private String description;
    private Boolean isActive;
    private Date lastUpdateTime;
    
    ···
}   
```


## 表

- Role

```
CREATE TABLE role (
  id               INT         NOT NULL AUTO_INCREMENT PRIMARY KEY,
  name             VARCHAR(50) NOT NULL UNIQUE,
  description      VARCHAR(100),
  is_active        BOOLEAN     NOT NULL DEFAULT TRUE,
  last_update_time TIMESTAMP            DEFAULT current_timestamp ON UPDATE current_timestamp
);

INSERT INTO role (name, description, is_active, last_update_time)
VALUES ('ROLE_ADMIN', 'Administrator', TRUE, current_timestamp);
INSERT INTO role (name, description, is_active, last_update_time)
VALUES ('ROLE_USER', 'User', TRUE, current_timestamp);
```

- Menu
```
CREATE TABLE menu (
  id               INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  value            VARCHAR(100) NOT NULL,
  display_value    VARCHAR(100) NOT NULL,
  url              VARCHAR(100) NOT NULL,
  category         INT,
  description      VARCHAR(100),
  is_active        BOOLEAN      NOT NULL DEFAULT TRUE,
  last_update_time TIMESTAMP             DEFAULT current_timestamp ON UPDATE current_timestamp
);

INSERT INTO menu (value, display_value, url, description, is_active, last_update_time)
VALUES ('/admin/dashboard', 'Admin Dashboard', '/admin/dashboard', 'Admin Dashboard', TRUE, current_timestamp);

INSERT INTO menu (value, display_value, url, description, is_active, last_update_time)
VALUES ('/admin/profile', 'Admin Profile', '/admin/profile', 'Admin Profile', TRUE, current_timestamp);


INSERT INTO menu (value, display_value, url, description, is_active, last_update_time)
VALUES ('/user/dashboard', 'User Dashboard', '/user/dashboard', 'User Dashboard', TRUE, current_timestamp);

INSERT INTO menu (value, display_value, url, description, is_active, last_update_time)
VALUES ('/user/profile', 'User Profile', '/user/profile', 'User Profile', TRUE, current_timestamp);

```
- RoleMenuXref
```
CREATE TABLE role_menu_xref (
  id               INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  role_id          INT NOT NULL,
  menu_id          INT NOT NULL,
  last_update_time TIMESTAMP    DEFAULT current_timestamp ON UPDATE current_timestamp,
  CONSTRAINT FOREIGN KEY fk_role_menu_xref_role_id (role_id) REFERENCES role (id),
  CONSTRAINT FOREIGN KEY fk_role_menu_xref_menu_id(menu_id) REFERENCES menu (id),
  CONSTRAINT UNIQUE (role_id, menu_id)
);

INSERT role_menu_xref (role_id, menu_id, last_update_time)
VALUES (1, 1, current_timestamp);

INSERT role_menu_xref (role_id, menu_id, last_update_time)
VALUES (1, 2, current_timestamp);

INSERT role_menu_xref (role_id, menu_id, last_update_time)
VALUES (2, 3, current_timestamp);

INSERT role_menu_xref (role_id, menu_id, last_update_time)
VALUES (2, 4, current_timestamp);


```


------------------

##  Collection 一个查询调用另一个查询实现的嵌套

```
        <resultMap id="BaseRoleResultMap" type="cn.com.hellowood.springsecurity.model.RoleModel">
        <id column="id" property="id" jdbcType="INTEGER"/>
        <result column="name" property="name" jdbcType="VARCHAR"/>
        <result column="is_active" property="isActive" jdbcType="BOOLEAN"/>
        <result column="description" property="description" jdbcType="VARCHAR"/>
        <result column="last_update_time" property="lastUpdateTime" jdbcType="TIMESTAMP"/>
        <collection property="menus" ofType="cn.com.hellowood.springsecurity.model.menus"
                    javaType="java.util.ArrayList" select="getMenus"
                    column="id">
        </collection>
    </resultMap>


    <resultMap id="BaseMenuResultMap" type="cn.com.hellowood.springsecurity.model.MenuModel">
        <id column="id" property="id" jdbcType="INTEGER"/>
        <result column="value" property="value" jdbcType="VARCHAR"/>
        <result column="display_value" property="displayValue" jdbcType="VARCHAR"/>
        <result column="url" property="url" jdbcType="VARCHAR"/>
        <result column="category" property="category" jdbcType="INTEGER"/>
        <result column="description" property="description" jdbcType="VARCHAR"/>
        <result column="is_active" property="isActive" jdbcType="BIT"/>
        <result column="last_update_time" property="lastUpdateTime" jdbcType="TIMESTAMP"/>
    </resultMap>
    
    <select id="getRoles" parameterType="java.lang.Integer" resultMap="BaseRoleResultMap">
        SELECT
            id,
            name,
            is_active,
            description,
            last_update_time
        FROM role
    </select>
    
    <select id="getMenus" parameterType="java.lang.Integer" resultMap="BaseMenuResultMap">
        SELECT
            m.id,
            m.value,
            m.display_value,
            m.url,
            m.category,
            m.description,
            m.is_active,
            m.last_update_time
        FROM menu m
            LEFT JOIN
            role_menu_xref rmx
                ON m.id = rmx.menu_id
        WHERE role_id = #{roleId, jdbcType=INTEGER}
    </select>

```

> 此时，调用 `getRoles()` 方法就可以通过嵌套查询同时查找 Role 属性了

```
        <collection property="menus" ofType="cn.com.hellowood.springsecurity.model.menus"
                    javaType="java.util.ArrayList" select="getMenus"
                    column="id">
        </collection>
```
> - collection : 一个复杂的类型关联，许多结果将映射为这种类型
> - property : 这是关联的 JavaBean 中的属性名， 在 RoleModel 中对应 `private List<MenuModel> menus;`
> - javaType : property 属性对应的集合类型
> - ofType : property 集合中的泛型，在 RoleModel 中是 MenuModel
> - column : RoleModel 的 id ，作为参数传入被调用的 Select 语句
> - select : 另外一个映射语句的 ID

--------------

## Collection 同一个查询映射到属性的嵌套 
> 如果再一个查询中可以直接查询到所需要的数据，但是需要映射到该对象的属性上，则可以使用该方式

```
    <resultMap id="BaseRoleResultMap" type="cn.com.hellowood.springsecurity.model.RoleModel">
        <id column="id" property="id" jdbcType="INTEGER"/>
        <result column="name" property="name" jdbcType="VARCHAR"/>
        <result column="is_active" property="isActive" jdbcType="BOOLEAN"/>
        <result column="description" property="description" jdbcType="VARCHAR"/>
        <result column="last_update_time" property="lastUpdateTime" jdbcType="TIMESTAMP"/>
        <collection property="menus" ofType="cn.com.hellowood.springsecurity.model.MenuModel"
                    javaType="java.util.ArrayList">
            <id column="id" property="id" jdbcType="INTEGER"/>
            <result column="value" property="value" jdbcType="VARCHAR"/>
            <result column="display_value" property="displayValue" jdbcType="VARCHAR"/>
            <result column="url" property="url" jdbcType="VARCHAR"/>
            <result column="category" property="category" jdbcType="INTEGER"/>
            <result column="description" property="description" jdbcType="VARCHAR"/>
            <result column="is_active" property="isActive" jdbcType="BIT"/>
            <result column="last_update_time" property="lastUpdateTime" jdbcType="TIMESTAMP"/>
        </collection>
    </resultMap>
    
    <select id="getRoles" parameterType="java.lang.Integer" resultMap="BaseRoleResultMap">
        SELECT
            r.id,
            r.name,
            r.description,
            r.is_active,
            r.last_update_time,
            m.id,
            m.value,
            m.display_value,
            m.url,
            m.category,
            m.description,
            m.is_active,
            m.last_update_time
        FROM role r
            LEFT JOIN role_menu_xref rmx
                ON r.id = rmx.role_id
            LEFT JOIN menu m
                ON m.id = rmx.menu_id
        WHERE r.id = #{roleId,jdbcType=INTEGER}
    </select>
```