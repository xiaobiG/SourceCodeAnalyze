# 基本使用

## Java调用JS

在网页中使用 Js 定义提供给 Java 访问的方法，就像普通方法定义一样，如：

```java
<script type="text/javascript">
    function javaCallJs(message){
        alert(message);
    }
</script>
```

在 Java 代码中按照 "javascript:XXX" 的 Url 格式使用 WebView 加载访问即可：

```java
mWebView.loadUrl("javascript:javaCallJs(" + "'Message From Java'" + ")");
```

## JS 调用 Java

在 Java 对象中定义 Js 访问的方法，如：

```java
@JavascriptInterface
public void jsCallJava(String message){
    Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
}
```

将提供给 Js 访问的接口内容所属的 Java 对象注入 WebView 中：

```java
mWebView.addJavascriptInterface(MainActivity.this, "main");
```

Js 按照指定的接口名访问 Java 代码，有如下两种写法：

```java
<button type="button" onClick="javascript:main.jsCallJava('Message From Js')" >Js Call Java</button>

<!--<button type="button" onClick="window.main.jsCallJava('Message From Js')" >Js Call Java</button>-->
```

## 注意事项

1. 使用 loadUrl() 方法实现 Java 调用 Js 功能时，必须放置在主线程中，否则会发生崩溃异常。
2. Js 调用 Java 方法时，不是在主线程 (Thread Name：main) 中运行的，而是在一个名为 JavaBridge 的线程中执行的
3. 代码混淆时，记得保持 JavascriptInterface 内容，在 proguard 文件中添加如下类似规则 (有关类名按需修改)：

```java
keepattributes *Annotation*
keepattributes JavascriptInterface
-keep public class com.mypackage.MyClass$MyJavaScriptInterface
-keep public class * implements com.mypackage.MyClass$MyJavaScriptInterface
-keepclassmembers class com.mypackage.MyClass$MyJavaScriptInterface { 
    <methods>; 
}
```

# 基本原理

Mark