---
title: Gauge中执行数据库测试
date: 2018-01-01 11:34:11
tags:
    - Java
    - Gauge
    - Test
categories: 
    - Java
    - Gauge
    - Test
---
> 使用Gauge对数据库的增删改查进行测试


----------
## 打开数据库连接
- .spec文件

```Markdown
    * open connection before crud
```
- .java文件

```java
    private Connection connection;
    private PreparedStatement statement;

    @Step("open connection before crud")
    public void openConnection() {
        try {
            String driver = "com.mysql.jdbc.Driver";
            String url = "jdbc:mysql://119.29.99.89:3306/springboot?useSSL=false";
            String username = "victor";
            String password = "Victor123456";

            Class.forName(driver);
            connection = DriverManager.getConnection(url, username, password);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

----------
## insert
- .spec文件

```Markdown
    ## insert
    insert record to database
     tags: crud,insert
    
    * insert new record named "Gauge",sex is "Male",age is "25"

```

- .java文件

```java
    @Step("insert new record named <Gauge>,sex is <Male>,age is <25>")
    public void insert(String name, String sex, Integer age) {
        try {
            statement = connection.prepareStatement("insert into user(username,sex,age) values (?,?,?)");
            statement.setString(1, name);
            statement.setString(2, sex);
            statement.setInt(3, age);

            int result = statement.executeUpdate();

            assertTrue(result > 0);

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
```


----------
## select
- .spec文件

```Markdown
    ## query
    query all records from database
     tags: crud,select
    
    * query all records
```

- .java文件

```java
    @Step("query all records")
    public void query() {
        try {
            statement = connection.prepareStatement("SELECT * FROM user");
            ResultSet resultSet = statement.executeQuery();
            while (resultSet.next()) {
                    StringBuffer userInfo = new StringBuffer();
                    userInfo.append("ID:" + resultSet.getString("id"));
                    userInfo.append("\t\tUsername:" + resultSet.getString("username"));
                    userInfo.append("\t\tSex:" + resultSet.getString("sex"));
                    userInfo.append("\t\tAge:" + resultSet.getString("age"));
                    userInfo.append("\t\tSchool:" + resultSet.getString("school"));
                    userInfo.append("\t\tMajor:" + resultSet.getString("major"));
                    userInfo.append("\t\tAddress:" + resultSet.getString("address"));
                    Gauge.writeMessage(userInfo.toString());
            }

            resultSet.last();
            int rowCount = resultSet.getRow();
            resultSet.close();

            assertTrue(rowCount > 0);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
```

----------
## update
- .spec文件

```
    ## update
    update record
     tags: crud,update
     
    * update record sex to "Female" which named "Gauge"
```

- .java文件

```java
    @Step("update record sex to <Female> which named <Gauge>")
    public void update(String sex, String name) {
        try {
            statement = connection.prepareStatement("UPDATE user SET sex=? WHERE username=?");
            statement.setString(1, sex);
            statement.setString(2, name);
            int resultNum = statement.executeUpdate();
            assertTrue(resultNum > 0);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
```


----------
## delete
- .spec文件

```Markdown
    ## delete
    delete record
     tags: crud,delete
    
    * delete the record which named "Gauge"

```

- .java文件

```java
    @Step("delete the record which named <Guage>")
    public void delete(String name) {
        try {
            statement = connection.prepareStatement("DELETE FROM user WHERE username=?");
            statement.setString(1, name);
            int resultNum = statement.executeUpdate();
            assertTrue(resultNum > 0);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
```


----------
## 关闭连接
- .spec文件

```Markdown
    * close connection after crud
```

- .java文件

```java
    @Step("close connection after crud")
    public void closeConnection() {
        try {
            statement.close();
            connection.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
```


----------
- 需要把Jar包放在项目的lib目录下
- 使用了[MySQL驱动](https://dev.mysql.com/downloads/file/?id=465644)
- [下载项目](http://download.csdn.net/detail/u013360850/9760640)

