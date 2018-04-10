---
title: Gauge 中Step使用别名
date: 2018-01-01 11:29:48
tags:
    - Java
    - Gauge
    - Test
categories: 
    - Java
    - Gauge
    - Test
---
所有内容均来自[Gauge官网文档](http://getgauge.io/documentation/user/current/advanced_readings/step_alias.html)


----------


##在执行的过程中，可能需要使用不同的名称来执行同样的操作，所以使用别名来区分


----------
- 在这个Scenario中，第一步和第三步是相同的操作，但是表示的方式不同

```
    User Creation
    =============
    Multiple Users
    --------------
    * Create a user "user 1"
    * Verify "user 1" has access to dashboard
    * Create another user "user 2"
    * Verify "user 2" has access to dashboard
```
- 使用别名即可解决这个问题：

```
    public class Users {
    
        @Step({"Create a user <user_name>", "Create another user <user_name>"})
        public void createUser(String user_name) {
            // create user user_name
        }
    }
```


----------

- 在这个两个Scenario中，发送邮件的操作是相同的
```
    User Creation
    -------------
    * User creates a new account
    * A "welcome" email is sent to the user
    
    Shopping Cart
    -------------
    * User checks out the shopping cart
    * Payment is successfully received
    * An email confirming the "order" is sent
```
- 使用别名：

```
    public class Users {
    
        @Step({"A <email_type> email is sent to the user", "An email confirming the <email_type> is sent"})
        public void sendEmail(String email_type) {
            // Send email of email_type
        }
    }
```