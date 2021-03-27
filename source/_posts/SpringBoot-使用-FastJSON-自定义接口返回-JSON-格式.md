---
title: SpringBoot 使用 FastJSON 自定义接口返回 JSON 格式
date: 2018-04-10 18:50:34
tags:
    - Java
    - SpringBoot 
    - FastJSON
categories: 
    - Java
    - SpringBoot
    - FastJSON
---

# SpringBoot 使用 FastJSON 自定义接口返回 JSON 格式

在 SpringBoot 中如果想要自定义接口返回的值格式，可以通过重写 `WebMvcConfigurerAdapter` 类的 `configureMessageConverters` 方法实现

- 添加依赖

```
compile('com.alibaba:fastjson:1.2.46')
```

- MessageConverter.java

```
import com.alibaba.fastjson.serializer.SerializerFeature;
import com.alibaba.fastjson.support.config.FastJsonConfig;
import com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

import java.util.List;

@Component
public class MessageConverter extends WebMvcConfigurerAdapter {

    private static final String DATE_TIME_PATTEN = "yyyy-MM-dd HH:mm:ss";

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        FastJsonHttpMessageConverter fastJsonHttpMessageConverter = new FastJsonHttpMessageConverter();

        FastJsonConfig fastJsonConfig = new FastJsonConfig();
        // 设置空值不返回
        fastJsonConfig.setSerializerFeatures(SerializerFeature.PrettyFormat);
        // 设置日期格式
        fastJsonConfig.setDateFormat(DATE_TIME_PATTEN);
        fastJsonHttpMessageConverter.setFastJsonConfig(fastJsonConfig);

        converters.add(fastJsonHttpMessageConverter);
    }
}
```

- 如果不想让某个列返回可以在属性上添加 `@JSONField(serialize = false)`
- 如果想让某个属性返回的值为指定的字符串：`@JSONField(name = "custom_field_name")`
