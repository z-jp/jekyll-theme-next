---
layout: post
title: try-with-resources Statement
category: Java
tags: Java
excerpt_separator: "`"
---
> The try-with-resources statement is a try statement that declares one or more resources.

在 JDK7 之前，打开一个文件并获取输入流是这样的：
````
try {
    //Statement A
    FileInputStream fis = new FileInputStream(FILE_PATH);
    //Statement B
    fis.close();
} catch (IOException e) {
    e.printStackTrace();
}
````
语句B处发生异常将时 fis 不能正常关闭资源，应该在 finally 中关闭：
````
FileInputStream fis = null;
try {
    //Statement A
    fis = new FileInputStream(FILE_PATH);
    //Statement B
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fis != null) {
        try {
            fis.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
````  
fis 在 try 块外声明，需要判断是否为 null ，同时close方法会抛异常，需要在 try 块中。这样下来，语法尤其啰嗦。 JDK7 中提供了 try-with-resouces 语句，即在 try 语句中声明资源对象，系统将会在语句完成时自动关闭资源。
````
try (FileInputStream fis = new FileInputStream(FILE_PATH)) {
    //Statement A
    //Statement B
} catch (IOException e) {
    e.printStackTrace();
}
````
- 可以在 try 中声明的资源必须实现 AutoCloseable 接口，而多数资源相关的类实现的 Closeable 接口就继承自 AutoCloseable
- 可以在 try 中声明多个资源，越早声明的对象越晚被关闭（后声明的资源往往依赖于之前创建的资源）
- try-with-resources 语句同样可以使用 finally 语句，catch 和 finally 语句在资源关闭后执行

回编译生成的 .class 文件可以看到最终还是使用了 finally 语句，所以 try-with-resources 语句只是一颗语法糖，但可以使代码简练很多。 JDK7-8 中对于一个已有的 final 或者 effective final 资源对象，必须创建副本才能在try块中引用
````
CloseableObject object = new CloseableObject();
try (CloseableObject finalObject = object){
    finalObject.doSomething();
}
````

在 JDK9 中可以直接引用
````
CloseableObject object = new CloseableObject();
try (object){
    object.doSomething();
}
````
实现方式同样是在 try 块中创建副本，只是改成由编译器创建...