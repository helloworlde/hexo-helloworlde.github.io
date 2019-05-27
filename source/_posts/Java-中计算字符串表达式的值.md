---
title: Java 中计算字符串表达式的值
date: 2018-08-25 10:55:46
tags:
    - Java
categories: 
    - Java 
---

# Java 中计算字符串表达式的值

> 在 Java 中计算字符串数值表达式可以用 `javax.script.ScriptEngine#eval(java.lang.String)`，通过调用 JavaScript 来计算


```java
import javax.script.ScriptEngine;
import javax.script.ScriptEngineManager;
import javax.script.ScriptException;

public class ExpressionCalculate {
    public static void main(String[] args) {
        ScriptEngineManager scriptEngineManager = new ScriptEngineManager();
        ScriptEngine scriptEngine = scriptEngineManager.getEngineByName("nashorn");
        String expression = "10 * 2 + 6 / (3 - 1)";

        try {
            String result = String.valueOf(scriptEngine.eval(expression));
            System.out.println(result);
        } catch (ScriptException e) {
            e.printStackTrace();
        }
    }
}
```
结果： 23