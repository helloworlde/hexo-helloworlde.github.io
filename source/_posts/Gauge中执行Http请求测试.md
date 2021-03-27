---
title: Gauge中执行Http请求测试
date: 2018-01-01 11:33:29
tags:
    - Java
    - Gauge
    - Test
categories: 
    - Java
    - Gauge
    - Test
---

> 通过Gauge执行自动化测试，测试Http请求
> 
- 通过Java发送Http 请求来测试服务器请求执行状态


----------


##GET请求
- .spec文件

```
    ## query user
    query all user
    tags: query,request,http
    
    * query user

```
- .java文件

``` java
    @Step("query user")
    public void queryUser() {

        String url = "http://119.29.99.89:8080/query";

        String result = sendGetRequest(url);

        //用到了FastJSON来处理返回的Json数据
        JSONArray users = JSON.parseArray(result);
        for (int i = 0; i < users.size(); i++) {
            UserModel user = JSON.parseObject(users.get(i).toString(), UserModel.class);
            Gauge.writeMessage(user.toString());
        }
        assertTrue(users != null && !users.isEmpty());
    }

    //发送一个GET请求
    private String sendGetRequest(String url) {
        String result = "";
        BufferedReader in = null;

        try {
            URL realUrl = new URL(url);
            URLConnection connection = realUrl.openConnection();

            connection.connect();

            in = new BufferedReader(new InputStreamReader(connection.getInputStream()));
            String line;
            while ((line = in.readLine()) != null) {
                result += line;
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (in != null) {
                    in.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return result;
    }
```


----------

##POST请求
- .spec文件

``` Markdown
    ## add user
    add new user
    tags: add,request,http
    
    * add users from table
    |username|sex   |age|school    |major |address   |
    |--------|------|---|----------|------|----------|
    |Angular |Male  |21 |大连理工大学|软件工程|辽宁省大连市|
    |JQuery  |Female|22 |大连交通大学|网络工程|辽宁省大连市|
    |Vue     |Female|23 |大连理工大学|软件工程|辽宁省大连市|
    |React   |Male  |24 |大连海事大学|网络工程|辽宁省大连市|
```
- .java文件

``` java
    @Step("add users from table <table>")
    public void addUser(Table userTable) {

        boolean result = true;
        String url = "http://119.29.99.89:8080/add";

        for (TableRow row : userTable.getTableRows()) {
            UserModel userModel = new UserModel();
            userModel.setUsername(row.getCell("username"));
            userModel.setSex(row.getCell("sex"));
            userModel.setAge(Integer.valueOf(row.getCell("age")));
            userModel.setSchool(row.getCell("school"));
            userModel.setMajor(row.getCell("major"));
            userModel.setAddress(row.getCell("address"));

            String requestString = JSON.toJSONString(userModel);

            String response = sendPostRequest(requestString, url);
            JSONObject responseMap = JSON.parseObject(response);
            result = result && responseMap.get("STATE").equals("SUCCESSED");
        }
        assertTrue(result);
    }

    //发送POST请求
     private String sendPostRequest(String requestString, String url) {
        PrintWriter out = null;
        BufferedReader in = null;
        String result = "";
        try {
            URL realUrl = new URL(url);
            URLConnection conn = realUrl.openConnection();

            conn.setRequestProperty("content-type", "application/json");
            conn.setDoOutput(true);
            conn.setDoInput(true);

            out = new PrintWriter(conn.getOutputStream());
            out.print(requestString);
            out.flush();

            in = new BufferedReader(new InputStreamReader(conn.getInputStream()));
            String line;
            while ((line = in.readLine()) != null) {
                result += line;
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (out != null) {
                    out.close();
                }
                if (in != null) {
                    in.close();
                }
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }
        return result;
    }
```


----------

- UserModel.java

``` java
    public class UserModel {
        private int id;
        private String username;
        private String sex;
        private int age;
        private String school;
        private String major;
        private String address;
    
        public UserModel() {
            super();
        }
    
        public UserModel(int id, String username, String sex, int age, String school, String major, String address) {
            super();
            this.id = id;
            this.username = username;
            this.sex = sex;
            this.age = age;
            this.school = school;
            this.major = major;
            this.address = address;
        }
    
        public int getId() {
            return id;
        }
    
        public void setId(int id) {
            this.id = id;
        }
    
        public String getUsername() {
            return username;
        }
    
        public void setUsername(String username) {
            this.username = username;
        }
    
        public String getSex() {
            return sex;
        }
    
        public void setSex(String sex) {
            this.sex = sex;
        }
    
        public int getAge() {
            return age;
        }
    
        public void setAge(int age) {
            this.age = age;
        }
    
        public String getSchool() {
            return school;
        }
    
        public void setSchool(String school) {
            this.school = school;
        }
    
        public String getMajor() {
            return major;
        }
    
        public void setMajor(String major) {
            this.major = major;
        }
    
        public String getAddress() {
            return address;
        }
    
        public void setAddress(String address) {
            this.address = address;
        }
    
        @Override
        public String toString() {
            return "UserModel{" +
                    "id=" + id +
                    ", username='" + username + '\'' +
                    ", sex='" + sex + '\'' +
                    ", age=" + age +
                    ", school='" + school + '\'' +
                    ", major='" + major + '\'' +
                    ", address='" + address + '\'' +
                    '}';
        }
    }

```


----------


- 需要把Jar包放在项目的lib目录下
- 使用了[阿里巴巴的fastjson](https://github.com/alibaba/fastjson) 
- [下载项目](http://download.csdn.net/detail/u013360850/9760640) 