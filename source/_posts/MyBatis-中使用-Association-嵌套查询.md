---
title: MyBatis 中使用 Association 嵌套查询
date: 2018-01-01 00:48:06
tags:
    - Java
    - MyBatis
categories: 
    - Java
    - MyBatis
---

> 当使用 MyBatis 进行查询的时候如果一个 JavaBean 中包含另一个 JavaBean 或者 Collection 时，可以通过 MyBatis 的嵌套查询来获取需要的结果;
> 以下以用户登录时的用户、角色和菜单直接的关系为例使用嵌套查询

------------

## JavaBean 
- UserModel
```
public class UserModel {
    private Integer id;
    private String username;
    private String password;
    private Boolean enabled;
    private Boolean locked;
    private Boolean expired;
    private RoleModel role;
    private List<MenuModel> menus;

    ···
}   
```
- RoleModel
```
public class RoleModel {
    private Integer id;
    private String name;
    private Boolean isActive;
    private String description;
    private Date lastUpdateTime;

    ···
}   
```

## 表
- User 
```
CREATE TABLE user (
  id               INT                  AUTO_INCREMENT PRIMARY KEY,
  username         VARCHAR(45) NOT NULL,
  password         VARCHAR(45) NOT NULL,
  enabled          BOOLEAN     NOT NULL DEFAULT TRUE,
  expired          BOOLEAN     NOT NULL DEFAULT TRUE,
  locked           BOOLEAN     NOT NULL DEFAULT TRUE,
  last_update_time TIMESTAMP   NOT NULL DEFAULT current_timestamp ON UPDATE current_timestamp,
  comment          VARCHAR(100)
);

INSERT INTO user (username, password, enabled, expired, locked, last_update_time)
VALUES ('admin', 'admin', TRUE, FALSE, FALSE, current_timestamp);
INSERT INTO user (username, password, enabled, expired, locked, last_update_time)
VALUES ('user', 'password', TRUE, FALSE, FALSE, current_timestamp);
```

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

- UserRoleXref
```
CREATE TABLE user_role_xref (
  id               INT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  user_id          INT       NOT NULL,
  role_id          INT       NOT NULL,
  last_update_time TIMESTAMP NOT NULL DEFAULT current_timestamp ON UPDATE current_timestamp,
  CONSTRAINT FOREIGN KEY fK_user_role_xref_user_id_user_id (user_id) REFERENCES user (id),
  CONSTRAINT FOREIGN KEY fk_user_role_xref_role_id_role_id (role_id) REFERENCES role (id)
);

INSERT INTO user_role_xref (user_id, role_id, last_update_time) VALUES (1, 1, CURRENT_TIMESTAMP);
INSERT INTO user_role_xref (user_id, role_id, last_update_time) VALUES (2, 2, CURRENT_TIMESTAMP);
```

------------------

##  一个查询调用另一个查询实现的嵌套

```
    <resultMap id="BaseUserModelResultMap" type="cn.com.hellowood.springsecurity.model.UserModel">
        <id column="id" property="id" javaType="java.lang.Integer" jdbcType="INTEGER"></id>
        <result column="username" property="username" javaType="java.lang.String" jdbcType="VARCHAR"></result>
        <result column="password" property="password" javaType="java.lang.String" jdbcType="VARCHAR"></result>
        <result column="enabled" property="enabled" javaType="java.lang.Boolean" jdbcType="BOOLEAN"></result>
        <result column="expired" property="expired" javaType="java.lang.Boolean" jdbcType="BOOLEAN"></result>
        <result column="locked" property="locked" javaType="java.lang.Boolean" jdbcType="BOOLEAN"></result>
        <association property="role" javaType="cn.com.hellowood.springsecurity.model.RoleModel"
                     column="id" select="getRoleByUserId">
        </association>
    </resultMap>

    <resultMap id="BaseRoleResultMap" type="cn.com.hellowood.springsecurity.model.RoleModel">
        <id column="id" property="id" jdbcType="INTEGER"/>
        <result column="name" property="name" jdbcType="VARCHAR"/>
        <result column="is_active" property="isActive" jdbcType="BOOLEAN"/>
        <result column="description" property="description" jdbcType="VARCHAR"/>
        <result column="last_update_time" property="lastUpdateTime" jdbcType="TIMESTAMP"/>
    </resultMap>

    <select id="getAllUsers" resultMap="BaseUserModelResultMap">
        SELECT
            id,
            username,
            password,
            enabled,
            expired,
            locked
        FROM user
    </select>

    <select id="getRoleByUserId" parameterType="java.lang.Integer"
            resultType="cn.com.hellowood.springsecurity.model.RoleModel">
        SELECT
            r.id,
            r.name,
            r.is_active,
            r.description,
            r.last_update_time
        FROM role r
            LEFT JOIN user_role_xref urx
                ON r.id = urx.role_id
        WHERE user_id = #{userId, jdbcType=INTEGER}
    </select>
```

> 此时，调用 `getAllUsers()` 方法就可以通过嵌套查询同时查找 Role 属性了

```
        <association property="role" javaType="cn.com.hellowood.springsecurity.model.RoleModel"
                     column="id" select="getRoleByUserId">
        </association>
```
> - association : 一个复杂的类型关联，许多结果将映射为这种类型
> - property : 这是关联的 JavaBean 中的属性名， 在 UserModel 中对应 `private RoleModel role;`
> - column : UserModel 的 id ，作为参数传入被调用的 Select 语句
> - select : 另外一个映射语句的 ID

--------------

## 同一个查询映射到属性的嵌套 
> 如果再一个查询中可以直接查询到所需要的数据，但是需要映射到该对象的属性上，则可以使用该方式

```
    <resultMap id="BaseUserModelResultMap" type="cn.com.hellowood.springsecurity.model.UserModel">
        <id column="id" property="id" javaType="java.lang.Integer" jdbcType="INTEGER"></id>
        <result column="username" property="username" javaType="java.lang.String" jdbcType="VARCHAR"></result>
        <result column="password" property="password" javaType="java.lang.String" jdbcType="VARCHAR"></result>
        <result column="enabled" property="enabled" javaType="java.lang.Boolean" jdbcType="BOOLEAN"></result>
        <result column="expired" property="expired" javaType="java.lang.Boolean" jdbcType="BOOLEAN"></result>
        <result column="locked" property="locked" javaType="java.lang.Boolean" jdbcType="BOOLEAN"></result>
        <association property="role" javaType="cn.com.hellowood.springsecurity.model.RoleModel">
            <id column="id" property="id" jdbcType="INTEGER"/>
            <result column="name" property="name" jdbcType="VARCHAR"/>
            <result column="is_active" property="isActive" jdbcType="BOOLEAN"/>
            <result column="description" property="description" jdbcType="VARCHAR"/>
            <result column="last_update_time" property="lastUpdateTime" jdbcType="TIMESTAMP"/>
        </association>
    </resultMap>

    <select id="getAllUsers" resultMap="BaseUserModelResultMap">
        SELECT
            u.id,
            u.username,
            u.password,
            u.enabled,
            u.expired,
            u.locked,
            r.id,
            r.name,
            r.is_active,
            r.description,
            r.last_update_time
        FROM user u
            LEFT JOIN user_role_xref ur
                ON u.id = ur.user_id
            LEFT JOIN role r
                ON ur.role_id = r.id
    </select>


```