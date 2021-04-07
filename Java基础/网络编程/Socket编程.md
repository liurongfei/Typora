## InetAddress



## URL类

```java
//使用URL读取网页内容
//创建一个URL实例
URL url =new URL("http://www.baidu.com");
InputStream is = url.openStream();//通过openStream方法获取资源的字节输入流
InputStreamReader isr =newInputStreamReader(is,"UTF-8");//将字节输入流转换为字符输入流,如果不指定编码，中文可能会出现乱码
BufferedReader br =newBufferedReader(isr);//为字符输入流添加缓冲，提高读取效率
String data = br.readLine();//读取数据
while(data!=null){
System.out.println(data);//输出数据
data = br.readerLine();
}
br.close();
isr.colose();
is.close();
```



## TCP编程

![img](C:\Environment\Github\Typora\Java基础\网络编程\740688-20150907234728090-211300057.jpg)