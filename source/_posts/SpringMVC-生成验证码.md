---
title: SpringMVC 生成验证码
date: 2018-01-01 01:04:16
tags:
    - Java
    - SrpingMVC 
categories: 
    - Java
    - SrpingMVC 
---

> 使用 Google kaptcha 为 SpringMVC Maven 项目生成验证码

##1 添加依赖

```
        <dependency>
            <groupId>com.github.penggle</groupId>
            <artifactId>kaptcha</artifactId>
            <version>2.3.2</version>
        </dependency>
```

## 2 配置文件中添加验证码生成器Bean

```
    <!--图片验证码-->
    <bean id="captchaProducer" class="com.google.code.kaptcha.impl.DefaultKaptcha">
        <property name="config">
            <bean class="com.google.code.kaptcha.util.Config">
                <constructor-arg>
                    <props>
                        <prop key="kaptcha.border">no</prop>
                        <prop key="kaptcha.image.width">120</prop>
                        <prop key="kaptcha.session.key">code</prop>
                        <prop key="kaptcha.textproducer.font.color">blue</prop>
                        <prop key="kaptcha.textproducer.font.size">40</prop>
                        <prop key="kaptcha.textproducer.char.length">4</prop>
                    </props>
                </constructor-arg>
            </bean>
        </property>
    </bean>
```
- 配置项


|属性         | 作用     | 说明   |
|---------------|--------|------|
|kaptcha.border | 是否有边框 | 默认为true  我们可以自己设置yes，no  |
|kaptcha.border.color  | 边框颜色 |  默认为Color.BLACK  |
|kaptcha.border.thickness  |边框粗细度|  默认为1  |
|kaptcha.producer.impl  | 验证码生成器 | 默认为DefaultKaptcha  |
|kaptcha.textproducer.impl  | 验证码文本生成器 | 默认为DefaultTextCreator  |
|kaptcha.textproducer.char.string  | 验证码文本字符内容范围  |默认为abcde2345678gfynmnpwx|  
|kaptcha.textproducer.char.length  | 验证码文本字符长度 | 默认为5  |
|kaptcha.textproducer.font.names |   验证码文本字体样式 | 默认为new Font("Arial", 1, fontSize), new Font("Courier", 1, fontSize)  |
|kaptcha.textproducer.font.size |  验证码文本字符大小 | 默认为40  |
|kaptcha.textproducer.font.color | 验证码文本字符颜色 | 默认为Color.BLACK  |
|kaptcha.textproducer.char.space | 验证码文本字符间距  |默认为2  |
|kaptcha.noise.impl |   验证码噪点生成对象 | 默认为DefaultNoise | 
|kaptcha.noise.color  | 验证码噪点颜色 |  默认为Color.BLACK  |
|kaptcha.obscurificator.impl |  验证码样式引擎 | 默认为WaterRipple | 
|kaptcha.word.impl  | 验证码文本字符渲染 |  默认为DefaultWordRenderer  |
|kaptcha.background.impl |  验证码背景生成器 |  默认为DefaultBackground | 
|kaptcha.background.clear.from |  验证码背景颜色渐进 |  默认为Color.LIGHT_GRAY  |
|kaptcha.background.clear.to  | 验证码背景颜色渐进  | 默认为Color.WHITE | 
|kaptcha.image.width  | 验证码图片宽度|  默认为200  |
|kaptcha.image.height | 验证码图片高度 | 默认为50|


## 3 页面添加验证码图片和输入框

```
            <div class="col-md-12">
                <div class="col-md-7 form-control"  style="float:left; width: 60%;">
                    <input type="text" id="validateCode" name="validateCode" placeholder="验证码" >
                </div>
                <div class="col-md-3" style="float: right;overflow: visible !important;">
                    <img src="./loadValidateCode" id="validateCodeImage" name="validateCodeImage"
                         style="width: 100px;height: 35px;"  onclick="loadValidateCode()" >
                </div>
            </div>
```
## 4 页面添加刷新验证码

```
    // 加载验证码
    function loadValidateCode() {
        var time = new Date().getTime();
        $("#validateCodeImage").attr('src', './loadValidateCode')
    }
```

## 5 后台添加生成验证码
 -  导入包
```
    import com.google.code.kaptcha.Producer;
```

- 生成方法

```Java
    @Autowired
    private Producer captchaProducer;

    private final String VALIDATE_CODE = "VALIDATE_CODE";

    private final String EXPIRE_TIME = "EXPIRE_TIME";
    
    @RequestMapping(value = "/loadValidateCode", method = RequestMethod.GET)
    public void loadValidateCode(HttpServletRequest request, HttpServletResponse response) {
        try {
            HttpSession session = request.getSession();

            // 设置清除浏览器缓存
            response.setDateHeader("Expires", 0);
            response.setHeader("Cache-Control", "no-store, no-cache, must-revalidate");
            response.addHeader("Cache-Control", "post-check=0, pre-check=0");
            response.setHeader("Pragma", "no-cache");
            response.setContentType("image/png");

            // 验证码一分钟内有效
            long expireTime = System.currentTimeMillis() + 60000;
            
            // 将验证码放到session中
            String validateCode = captchaProducer.createText();
            session.setAttribute(VALIDATE_CODE, Utils.encodeBase64(validateCode));//将加密后的验证码放到session中，确保安全
            session.setAttribute(EXPIRE_TIME, expireTime);

            // 输出验证码图片
            BufferedImage bufferedImage = captchaProducer.createImage(validateCode);
            ServletOutputStream out = response.getOutputStream();
            ImageIO.write(bufferedImage, "png", out);
            out.flush();
            out.close();

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```


## 6 登录时校验验证码

```
    @RequestMapping(value = "/login", method = RequestMethod.POST)
    public @ResponseBody String login(String username, String password, String validateCode) {
        
        // 校验验证码是否有效
        String currentValidateCode = String.valueOf(request.getSession().getAttribute(VALIDATE_CODE));
        if (System.currentTimeMillis() > Long.parseLong(String.valueOf(request.getSession().getAttribute(EXPIRE_TIME)))) {
            return JSON.toJSONString("验证码已过期，请重试");
        }

        // 校验验证码
        String currentValidateCode = String.valueOf(request.getSession().getAttribute(VALIDATE_CODE));
        if (StringUtils.isEmpty(validateCode) || validateCode.length() != 4 ||
                !Utils.encodeBase64(validateCode).equals(currentValidateCode)) {
            return JSON.toJSONString("验证码错误");
        }
 }       
```

## 7 Base64加密

```
    public static String encodeBase64(String str) {
        sun.misc.BASE64Encoder base64Encode = new BASE64Encoder();
        return base64Encode.encode(str.getBytes());
    }
```