RPC(Remote Procedure Call)
---------------

## RPC 是什么？
Remote Procedure Call:
```
In distributed computing, a remote procedure call (RPC) is when a computer program causes a procedure (subroutine) to execute in another address space (commonly on another computer on a shared network), which is coded as if it were a normal (local) procedure call, without the programmer explicitly coding the details for the remote interaction.
@Wiki
```
比如说，你的程序调用了一个函数，一般来说这个函数是在你这个程序本身定义和实现的。可是现在，这个函数的定义和实现却在另一个进程或者另一台电脑上，表面上你是直接对函数进行了调用，实际上调用时通过了进程间通信或者网络通信的。这种就叫做RPC。

RPC属于一种request/response协议，就是客户端发起一个请求，服务器端返回一个响应。其本身可以定义一些通信格式等，比如说JSON, XML, 或者binary等。

再来一个stackoverflow上的回答：
```
An RPC framework in general is a set of tools that enable the programmer to call a piece of code in a remote process, be it on a different machine or just another process on the same machine.
@ [stackoverflow](http://stackoverflow.com/questions/20653240/what-is-rpc-framework-and-apache-thrift)
```

## Thrift
先来看看官方定义：
```
Thrift is a software library and set of code-generation tools devel- oped at Facebook to expedite development and implementation of efficient and scalable backend services.
@《Thrift: Scalable Cross-Language Services Implementation》
```
Thrift是一个软件库(software library)以及一个代码生成工具的集合。由facebook提出，用来加速开发高效率和可扩展的后端服务(service)。
它解决的是什么问题呢？
在facebook，有很多个后端的service，然后每个service可能使用的开发语言并不一样。Service之间需要进行通信，于是就需要定义一套通信的传输标准，走网络的话，传输协议基本就是TCP或者HTTP了，而数据包里面的数据格式，则需要另一套标准。Thrift就提供了这一套标准，其传输的数据格式也可以进行选择，可以是JSON或者Binary。定义好标准之后，然后每个语言都需要实现一些代码来进行传输，这一部分，从功能上讲是重复的，而且可以单独提取出来。于是Thrift就提供了一个工具，可以针对不同的语言生成用于传输数据的代码。比如，你用java，那么用Thrift就可以生成java语言编写的发起Request和接收Response的代码，生成之后，业务代码就可以直接调用了。

Thrift可以实现RPC，怎么个实现法呢？Remote Procedure Call，这里的Procedure就是指一段代码，在编程里面，一段代码抽象一下，就是一个函数。Remote Procedure Call在某种程度上，可以称为Remote Method Call。既然是函数，自然有函数三件套：函数名，返回值，参数。实现RPC，自然要实现对函数的定义。而Thrift通过定义一个.thrift文件格式，实现了这种约束。通过后面的例子，可以略见一斑。

## Apache Thrift
#### 同样先来看看官方定义：
```
The Apache Thrift software framework, for scalable cross-language services development, combines a software stack with a code generation engine to build services that work efficiently and seamlessly between C++, Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, JavaScript, Node.js, Smalltalk, OCaml and Delphi and other languages.
@ [Apache Thrift](https://thrift.apache.org/)
```
其实跟上面的Thrift差不多，也是一个库(software stack)和一个代码生成工具。为什么呢？因为这个东西本来是facebook先搞出来的，后面贡献给了Apache，所以两个其实差不多咯。

##### 那么怎么使用呢？
大概步骤有三步：
* 写一个.thrift文件，定义方法
* 使用thrift工具产生代码
* 引入thrift library
* 编写客户端和服务器端代码

接下来用一个例子来解释：
首先定义一个.thrift文件service.thrift:
```
```

然后定义另一个share.thrift:
```
```

然后使用命令生成java代码：
```
thrift -r --gen java service.thrift
```

这会生成两个文件件：shared和tutorial。之后建立一个gradle工程，将shared和tutorial拷贝到`src/main/java`下面。
添加依赖，即在build.gradle里面的`dependencies`部分加上thrift的依赖：
```
dependencies {
    compile('org.apache.thrift:libthrift:0.9.3')
    testCompile group: 'junit', name: 'junit', version: '4.11'
}
```

然后添加一个package，这里是`li.koly`。然后添加client和server：
```
package li.koly;

import org.apache.thrift.TException;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;
import shared.SharedStruct;
import tutorial.Calculator;
import tutorial.InvalidOperation;
import tutorial.Operation;
import tutorial.Work;

public class JavaClient {
    public static void main(String[] args) {

        try {
            TTransport transport;
            transport = new TSocket("localhost", 9090);
            transport.open();
            TProtocol protocol = new TBinaryProtocol(transport);
            Calculator.Client client = new Calculator.Client(protocol);

            perform(client);

            transport.close();
        } catch (TException x) {
            x.printStackTrace();
        }
    }

    private static void perform(Calculator.Client client) throws TException {
        client.ping();
        System.out.println("ping()");

        int sum = client.add(1, 1);
        System.out.println("1+1=" + sum);

        Work work = new Work();

        work.op = Operation.DIVIDE;
        work.num1 = 1;
        work.num2 = 0;
        try {
            int quotient = client.calculate(1, work);
            System.out.println("Whoa we can divide by 0");
        } catch (InvalidOperation io) {
            System.out.println("Invalid operation: " + io.why);
        }

        work.op = Operation.SUBTRACT;
        work.num1 = 15;
        work.num2 = 10;
        try {
            int diff = client.calculate(1, work);
            System.out.println("15-10=" + diff);
        } catch (InvalidOperation io) {
            System.out.println("Invalid operation: " + io.why);
        }

        SharedStruct log = client.getStruct(1);
        System.out.println("Check log: " + log.value);
    }
}
```
这个是client，其中重点请参考注释。
下面是server。
```
package li.koly;

import org.apache.thrift.server.TServer;
import org.apache.thrift.server.TServer.Args;
import org.apache.thrift.server.TSimpleServer;
import org.apache.thrift.server.TThreadPoolServer;
import org.apache.thrift.transport.TSSLTransportFactory;
import org.apache.thrift.transport.TServerSocket;
import org.apache.thrift.transport.TServerTransport;
import org.apache.thrift.transport.TSSLTransportFactory.TSSLTransportParameters;

// Generated code
import tutorial.*;
import shared.*;

import java.util.HashMap;

public class JavaServer {

    public static CalculatorHandler handler;

    public static Calculator.Processor processor;

    public static void main(String [] args) {
        try {
            handler = new CalculatorHandler();
            processor = new Calculator.Processor(handler);

            Runnable simple = new Runnable() {
                public void run() {
                    simple(processor);
                }
            };

            new Thread(simple).start();
        } catch (Exception x) {
            x.printStackTrace();
        }
    }

    public static void simple(Calculator.Processor processor) {
        try {
            TServerTransport serverTransport = new TServerSocket(9090);
            TServer server = new TSimpleServer(new Args(serverTransport).processor(processor));

            // Use this for a multithreaded server
            // TServer server = new TThreadPoolServer(new TThreadPoolServer.Args(serverTransport).processor(processor));

            System.out.println("Starting the simple server...");
            server.serve();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

然后是具体的Calculator实现，也就是具体的业务实现了。
```

package li.koly;

import tutorial.*;
import shared.*;

import java.util.HashMap;

public class CalculatorHandler implements Calculator.Iface {

    private HashMap<Integer,SharedStruct> log;

    public CalculatorHandler() {
        log = new HashMap<Integer, SharedStruct>();
    }

    public void ping() {
        System.out.println("ping()");
    }

    public int add(int n1, int n2) {
        System.out.println("add(" + n1 + "," + n2 + ")");
        return n1 + n2;
    }

    public int calculate(int logid, Work work) throws InvalidOperation {
        System.out.println("calculate(" + logid + ", {" + work.op + "," + work.num1 + "," + work.num2 + "})");
        int val = 0;
        switch (work.op) {
            case ADD:
                val = work.num1 + work.num2;
                break;
            case SUBTRACT:
                val = work.num1 - work.num2;
                break;
            case MULTIPLY:
                val = work.num1 * work.num2;
                break;
            case DIVIDE:
                if (work.num2 == 0) {
                    InvalidOperation io = new InvalidOperation();
                    io.whatOp = work.op.getValue();
                    io.why = "Cannot divide by 0";
                    throw io;
                }
                val = work.num1 / work.num2;
                break;
            default:
                InvalidOperation io = new InvalidOperation();
                io.whatOp = work.op.getValue();
                io.why = "Unknown operation";
                throw io;
        }

        SharedStruct entry = new SharedStruct();
        entry.key = logid;
        entry.value = Integer.toString(val);
        log.put(logid, entry);

        return val;
    }

    public SharedStruct getStruct(int key) {
        System.out.println("getStruct(" + key + ")");
        return log.get(key);
    }

    public void zip() {
        System.out.println("zip()");
    }

}
```
首先启动server，然后启动client，就可以看到输出了。需要注意的是在client的perform函数里面，直接调用`add`函数，就跟调用本地函数的感觉一样，但是实际上是调用的另一个process的函数，这就是所谓的RPC了。


#### 版本演进怎么处理？
为什么需要版本演进呢？比如说有一天你希望增加或者修改一个service，但是同时又需要支持旧的service，这个时候怎么处理呢？

## 为什么使用RPC能得到更好的performance?
## Thrift与json，xml的比较
## 现在有哪些公司在使用Thrift?

参考资料：
[1] [Remote procedure call](https://en.wikipedia.org/wiki/Remote_procedure_call)
[2] [apache thrift](https://thrift.apache.org/) IPC(inter-process communication)
[3] [what is rpc framework and apache thrift](http://stackoverflow.com/questions/20653240/what-is-rpc-framework-and-apache-thrift)

