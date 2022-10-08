# JVM调优第一步，了解JVM常用命令行参数

- JVM的命令行参数参考：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html

- HotSpot参数分类
    >标准： - 开头，所有的HotSpot都支持
    >
    >非标准：-X 开头，特定版本HotSpot支持特定命令
    >
    >不稳定：-XX 开头，下个版本可能取消

    java -version

    java -X

    java -XX:+PrintFlagsFinal -version

试验用程序：

```java
import java.util.List;
import java.util.LinkedList;

public class HelloGC {
  public static void main(String[] args) {
    System.out.println("HelloGC!");
    List list = new LinkedList();
    for(;;) {
      byte[] b = new byte[1024*1024];
      list.add(b);
    }
  }
}
```

  1. 区分概念：内存泄漏memory leak，内存溢出out of memory
  2. java -XX:+PrintCommandLineFlags HelloGC
  3. java -Xmn10M -Xms40M -Xmx60M -XX:+PrintCommandLineFlags -XX:+PrintGC  HelloGC
    > PrintGC：粗略打印GC信息。
    > PrintGCDetails：打印详细的GC信息。
    > PrintGCTimeStamps：打印GC产生时系统的详细时间。
    > PrintGCCause：打印GC产生的原因。
  4. java -XX:+UseConcMarkSweepGC -XX:+PrintCommandLineFlags HelloGC
  5. java -XX:+PrintFlagsInitial 默认参数值
  6. java -XX:+PrintFlagsFinal 最终参数值
  7. java -XX:+PrintFlagsFinal | grep xxx 找到对应的参数
  8. java -XX:+PrintFlagsFinal -version |grep GC


# PS GC日志详解
每种垃圾回收器的日志格式是不同的！

PS日志格式






```shell

[root@localhost ~]# time ls
-                data        dokcer         HelloGC.java  root.crt  root.key    soft              test_for_stop.tar  webapp
anaconda-ks.cfg  Dockerfile  HelloGC.class  main.go       root.csr  server.key  test_for_run.tar  test.txt

real    0m0.018s
user    0m0.000s
sys     0m0.013s
```

> real：总共占用时间
> sys：内核态占用时间
> user：用户态占用时间