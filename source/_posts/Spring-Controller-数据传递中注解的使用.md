---
title: Spring Controller 数据传递中注解的使用
date: 2018-01-01 11:54:23
tags:
    - Java
    - SpringBoot 
categories: 
    - Java
    - SpringBoot
---
##根据处理Request的不同内容分为4类：
1. 处理`Request URI`部分的注解：`@PathVariable`
2. 处理`Request Header`部分的注解：`@RequestHeader`，`@CookieValue`
3. 处理`Request Body`部分的注解：`@RequestParam`，`@RequestBody`
4. 处理`Attribute`类型的注解：`@SessionAttribute`，`@ModelAttribute`

------------------------------------
##@PathVariable
- 当使用`@RequestMapping URI template`样式映射时，即`url/{param}`，这时`param`可以通过`@PathVariable`注解绑定它传过来的值到方法的参数上

```
    @Controller  
    public class RelativePathUriTemplateController {  
      
      @RequestMapping("/url/{param}")  
      public void getParams(@PathVariable String param) {      
        //....
      }  
    }  
```
----------------------------

##@RequestHeader
- 可以把`Request`请求的`Header`部分的值绑定到方法的参数上

```
    Host                    localhost:8080  
    Accept                  text/html,application/xhtml+xml,application/xml;q=0.9  
    Accept-Language         fr,en-gb;q=0.7,en;q=0.3  
    Accept-Encoding         gzip,deflate  
    Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7  
    Keep-Alive              300  
```

```
    @RequestMapping("/url")  
    public void displayHeaderInfo(@RequestHeader("Accept-Encoding") String encoding,  
                                  @RequestHeader("Keep-Alive") long keepAlive)  {  
      
      //...  
      
    }
```
> 把`request header`部分的 `Accept-Encoding`的值，绑定到参数`encoding`上了， `Keep-Alive header`的值绑定到参数`keepAlive`上。

--------------------------
##@CookieValue

- 可以把`RequestHeader`中关于`cookie`的值绑定到方法的参数上

```
@RequestMapping("/url")  
public void displayHeaderInfo(@CookieValue("JSESSIONID") String cookie)  {  
  
  //...  
  
} 
```

----------------------------

##@RequestBody
- 通常用来处理`Content-Type`不是`application/x-www-form-urlencoded`编码的内容，例如`application/json`，`application/xml`等
- 通过使用`HandlerAdapter`配置的`HttpMessageConverter`来解析`data body`，然后绑定到相应的`Bean`上
- 因为配置有`FormHttpMessageConverter`，所以也可以用来处理`application/x-www-form-urlencoded`的内容，处理完的结果放在一个`MultiValueMap<String,Stirng>`里

```
    @RequestMapping(value = "/url", method = RequestMethod.POST)  
    public void handle(@RequestBody String body) throws IOException {  
      //...
    } 
```
-----------------------------

##@ResponseBody
- 该注解用于将`Controller`的方法返回的对象，通过适当的`HttpMessageConverter`转换为指定格式的数据写入到`Response`对象的`body`数据区

```
    @ResponseBody
    @RequestMapping("/")
    public RedirectView root() {
        return new RedirectView("/index/index.html");
    }
```

-------------------------

## @SessionAttributes
- 用来绑定`HttpSession`中的`Attribute`对象的值

```
    @Controller  
    @RequestMapping("/editPet.do")  
    @SessionAttributes("pet")  
    public class EditPetForm {  
        // ...  
    }  
```

-------------------------

## @ModelAttribute
- 用于方法上时通常用来处理`@RequestMapping`之前，为请求绑定需要从后台查询的`Model`

```
    @ModelAttribute  
    public Account addAccount(@RequestParam String number) {  
        return accountManager.findAccount(number);  
    }  
```
> 这种方式实际的效果就是在调用`@RequestMapping`的方法之前，为`request`对象的`model`里`put（"account",Account）`

- 用于参数上时通过名称对应，把相应名称的值绑定到注解的参数`Bean`上，要绑定的值来源于
    - `@SessionAttributes`启用的`Attribute`对象
    - `@ModelAttribute`用于方法上时指定的`Model`对象
    - 以上两种情况都没有时，`new`一个需要绑定的`Bean`对象，然后把`Request`中按名称对应的方式把值绑定到`Bean`中
 
 

```
    @RequestMapping(value="/owners/{ownerId}/pets/{petId}/edit", method = RequestMethod.POST)  
    public String processSubmit(@ModelAttribute Pet pet) {  
         
    }  
```
> 首先查询 `@SessionAttributes`有无绑定的Pet对象，若没有则查询`@ModelAttribute`方法层面上是否绑定了Pet对象，若没有则将`URI template`中的值按对应的名称绑定到Pet对象的各属性上

