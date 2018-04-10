---
title: HTML中使用Ajax进行局部刷新页面
date: 2018-01-01 11:38:52
tags:
    - HTML
    - Jquery
categories: 
    - HTML
    - Jquery
---
#HTML中使用Ajax进行局部刷新页面，使用JS将数据发送到后台


----------


##1.在HTML页面中使用js脚本将请求数据发送给后台servlet
- 由按钮触发事件

```
<button id="select" onclick="queryInfos()">查询</button>
```

- 由js脚本对将数据发送到后台

```
    var req = new XMLHttpRequest();
    function queryInfos() {
        //设置传送方式，对应的servlet或action路径，是否异步处理
        req.open("POST", "/info/queryinfos", true);
        //如果设置数据传送方式为post，则必须设置请求头信息
        req.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
        //设置回调函数，当前操作完成后进行的操作
        req.onreadystatechange = callback;

        //Ajax请求发送的数据不是表单，需要拼接数据，格式和get方式一样
        var reqData = "ip=" + document.getElementById("ip").value;
        reqData += "&addr=" + document.getElementById("addr").value;
        reqData += "&time=" + document.getElementById("time").value;
        reqData += "&times=" + document.getElementById("times").value;

        //发送请求
        req.send(reqData);
    }
```

##2.后台获取数据进行处理，将结果返回给前台

- 后台进行处理
```
     protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
            //设置数据编码方式
            request.setCharacterEncoding("utf-8");
            response.setCharacterEncoding("utf-8");
            //设置数据类型
            response.setContentType("text/plain");
            //设置禁用缓存
            response.setHeader("Cache-control","no-cache");

            //获取从页面传递的参数
            String ip = request.getParameter("ip");
            String addr = request.getParameter("addr");
            String time = request.getParameter("time");
            String times = request.getParameter("times");

            /*
            * 执行操作
            * */

            //拼接返回的json数据
            StringBuilder jsonString = new StringBuilder();
            jsonString.append("{");
            jsonString.append("'size':2");

            jsonString.append(",'datas':[");

            jsonString.append("{");
            jsonString.append("'ip':'10.10.1.1',");
            jsonString.append("'addr':'辽宁葫芦岛',");
            jsonString.append("'time':'2016-10-24 16:00:23',");
            jsonString.append("'times':'10'");
            jsonString.append("}");

            jsonString.append(",{");
            jsonString.append("'ip':'192.168.110.111',");
            jsonString.append("'addr':'辽宁沈阳',");
            jsonString.append("'time':'2016-11-11 11:00:23',");
            jsonString.append("'times':'14'");
            jsonString.append("}");

            jsonString.append("]");

            jsonString.append("}");

            //获取输出流
            PrintWriter out = response.getWriter();
            //将数据返回页面
            out.write(jsonString.toString());
            out.flush();
            out.close();
        }
```

##3.返回处理结果，局部刷新页面

- 对返回操作进行处理
```
    //回调函数
    function callback() {
        //如果Ajax和request的状态都正确则进行操作
        if (req.readyState == 4 && req.status == 200) {
            //获取后台返回的数据
            var response = req.responseText;
            //进行对象化处理
            //加上圆括号的目的是迫使eval函数在处理JavaScript代码的时候强制将括号内的表达式转化为对象，而不是作为语句来执行
            //由于json是以"{}"的方式来开始以及结束的，在JS中，它会被当成一个语句块来处理，所以必须强制性的将它转换成一种表达式。
            var jsonobject = eval("(" + response + ")");
            //获取数据的长度
            var datasize = jsonobject.size;
            //获取json中的数据数据
            var datas = jsonobject.datas;

            //删除原有的table数据
            var table = document.getElementById("table");
            var infos = document.getElementById("tbody");
            table.removeChild(infos);
            //此处必须创建tbody，否则无法加入到table中
            infos = document.createElement("tbody");

            //生成新的table数据元素并添加到table中
            for (var i = 0; i < datas.length; i++) {
                var tr = document.createElement("tr");
                var td1 = document.createElement("td");
                var td2 = document.createElement("td");
                var td3 = document.createElement("td");
                var td4 = document.createElement("td");

                td1.innerHTML = datas[i].ip;
                td2.innerHTML = datas[i].addr;
                td3.innerHTML = datas[i].time;
                td4.innerHTML = datas[i].times;

                tr.appendChild(td1);
                tr.appendChild(td2);
                tr.appendChild(td3);
                tr.appendChild(td4);
                infos.appendChild(tr);
            }
            table.appendChild(infos);
        }
    }
```


----------


- HTML文件

```
<body>
<div class="title">
    <h1>Ajax异步查询，局部刷新页面</h1>
</div>
<div>
    <table class="select">
        <tr>
            <td class="td">IP:<input type="text" id="ip" name="ip" class="input"></td>
            <td class="td">地址:<input type="text" id="addr" name="addr" class="input"></td>
            <td class="td">时间:<input type="text" id="time" name="time" class="input"></td>
            <td class="td">次数:<input type="text" id="times" name="times" class="input"></td>
            <td class="td"> <button id="select" onclick="queryInfos()">查询</button></td>
        </tr>
    </table>
</div>
<table id="table" class="table" cellpadding="0" cellspacing="0" border="1">
    <tr>
        <td class="td">登录ip</td>
        <td class="td">登录地址</td>
        <td class="td">最后一次登录时间</td>
        <td class="td">登录次数</td>
    </tr>
    <tr>
        <td class="black" colspan="4"></td>
    </tr>
    <tbody id="tbody">
    <tr id="infos">
        <td>127.0.0.1</td>
        <td>辽宁大连</td>
        <td>2016-10-24 14:47:01</td>
        <td>123</td>
    </tr>
    </tbody>

</table>
</body>
```

----------
##[效果演示](http://project.hellowood.com.cn:8080/Ajax)
##[源码下载](http://download.csdn.net/detail/u013360850/9662587)