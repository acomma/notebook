全文转载自 [梁飞@ITeye](https://www.iteye.com/blog/user/javatar) 的博客 [RPC框架几行代码就够了](https://www.iteye.com/blog/javatar-1123915)。

转于自己在公司的Blog：

[http://pt.alibaba-inc.com/wp/experience_1330/simple-rpc-framework.html](http://pt.alibaba-inc.com/wp/experience_1330/simple-rpc-framework.html)

因为要给百技上实训课，让新同学们自行实现一个简易RPC框架，在准备PPT时，就想写个示例，发现原来一个RPC框架只要一个类，10来分钟就可以写完了，虽然简陋，也晒晒：

```java
/* 
 * Copyright 2011 Alibaba.com All right reserved. This software is the 
 * confidential and proprietary information of Alibaba.com ("Confidential 
 * Information"). You shall not disclose such Confidential Information and shall 
 * use it only in accordance with the terms of the license agreement you entered 
 * into with Alibaba.com. 
 */  
package com.alibaba.study.rpc.framework;  
  
import java.io.ObjectInputStream;  
import java.io.ObjectOutputStream;  
import java.lang.reflect.InvocationHandler;  
import java.lang.reflect.Method;  
import java.lang.reflect.Proxy;  
import java.net.ServerSocket;  
import java.net.Socket;  
  
/** 
 * RpcFramework 
 *  
 * @author william.liangf 
 */  
public class RpcFramework {  
  
    /** 
     * 暴露服务 
     *  
     * @param service 服务实现 
     * @param port 服务端口 
     * @throws Exception 
     */  
    public static void export(final Object service, int port) throws Exception {  
        if (service == null)  
            throw new IllegalArgumentException("service instance == null");  
        if (port <= 0 || port > 65535)  
            throw new IllegalArgumentException("Invalid port " + port);  
        System.out.println("Export service " + service.getClass().getName() + " on port " + port);  
        ServerSocket server = new ServerSocket(port);  
        for(;;) {  
            try {  
                final Socket socket = server.accept();  
                new Thread(new Runnable() {  
                    @Override  
                    public void run() {  
                        try {  
                            try {  
                                ObjectInputStream input = new ObjectInputStream(socket.getInputStream());  
                                try {  
                                    String methodName = input.readUTF();  
                                    Class<?>[] parameterTypes = (Class<?>[])input.readObject();  
                                    Object[] arguments = (Object[])input.readObject();  
                                    ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream());  
                                    try {  
                                        Method method = service.getClass().getMethod(methodName, parameterTypes);  
                                        Object result = method.invoke(service, arguments);  
                                        output.writeObject(result);  
                                    } catch (Throwable t) {  
                                        output.writeObject(t);  
                                    } finally {  
                                        output.close();  
                                    }  
                                } finally {  
                                    input.close();  
                                }  
                            } finally {  
                                socket.close();  
                            }  
                        } catch (Exception e) {  
                            e.printStackTrace();  
                        }  
                    }  
                }).start();  
            } catch (Exception e) {  
                e.printStackTrace();  
            }  
        }  
    }  
  
    /** 
     * 引用服务 
     *  
     * @param <T> 接口泛型 
     * @param interfaceClass 接口类型 
     * @param host 服务器主机名 
     * @param port 服务器端口 
     * @return 远程服务 
     * @throws Exception 
     */  
    @SuppressWarnings("unchecked")  
    public static <T> T refer(final Class<T> interfaceClass, final String host, final int port) throws Exception {  
        if (interfaceClass == null)  
            throw new IllegalArgumentException("Interface class == null");  
        if (! interfaceClass.isInterface())  
            throw new IllegalArgumentException("The " + interfaceClass.getName() + " must be interface class!");  
        if (host == null || host.length() == 0)  
            throw new IllegalArgumentException("Host == null!");  
        if (port <= 0 || port > 65535)  
            throw new IllegalArgumentException("Invalid port " + port);  
        System.out.println("Get remote service " + interfaceClass.getName() + " from server " + host + ":" + port);  
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class<?>[] {interfaceClass}, new InvocationHandler() {  
            public Object invoke(Object proxy, Method method, Object[] arguments) throws Throwable {  
                Socket socket = new Socket(host, port);  
                try {  
                    ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream());  
                    try {  
                        output.writeUTF(method.getName());  
                        output.writeObject(method.getParameterTypes());  
                        output.writeObject(arguments);  
                        ObjectInputStream input = new ObjectInputStream(socket.getInputStream());  
                        try {  
                            Object result = input.readObject();  
                            if (result instanceof Throwable) {  
                                throw (Throwable) result;  
                            }  
                            return result;  
                        } finally {  
                            input.close();  
                        }  
                    } finally {  
                        output.close();  
                    }  
                } finally {  
                    socket.close();  
                }  
            }  
        });  
    }  
  
}
```

用起来也像模像样：

### (1) 定义服务接口

```java
/* 
 * Copyright 2011 Alibaba.com All right reserved. This software is the 
 * confidential and proprietary information of Alibaba.com ("Confidential 
 * Information"). You shall not disclose such Confidential Information and shall 
 * use it only in accordance with the terms of the license agreement you entered 
 * into with Alibaba.com. 
 */  
package com.alibaba.study.rpc.test;  
  
/** 
 * HelloService 
 *  
 * @author william.liangf 
 */  
public interface HelloService {  
  
    String hello(String name);  
  
}
```

### (2) 实现服务

```java
/* 
 * Copyright 2011 Alibaba.com All right reserved. This software is the 
 * confidential and proprietary information of Alibaba.com ("Confidential 
 * Information"). You shall not disclose such Confidential Information and shall 
 * use it only in accordance with the terms of the license agreement you entered 
 * into with Alibaba.com. 
 */  
package com.alibaba.study.rpc.test;  
  
/** 
 * HelloServiceImpl 
 *  
 * @author william.liangf 
 */  
public class HelloServiceImpl implements HelloService {  
  
    public String hello(String name) {  
        return "Hello " + name;  
    }  
  
}
```

### (3) 暴露服务

```java
/* 
 * Copyright 2011 Alibaba.com All right reserved. This software is the 
 * confidential and proprietary information of Alibaba.com ("Confidential 
 * Information"). You shall not disclose such Confidential Information and shall 
 * use it only in accordance with the terms of the license agreement you entered 
 * into with Alibaba.com. 
 */  
package com.alibaba.study.rpc.test;  
  
import com.alibaba.study.rpc.framework.RpcFramework;  
  
/** 
 * RpcProvider 
 *  
 * @author william.liangf 
 */  
public class RpcProvider {  
  
    public static void main(String[] args) throws Exception {  
        HelloService service = new HelloServiceImpl();  
        RpcFramework.export(service, 1234);  
    }  
  
}
```

### (4) 引用服务

```java
/* 
 * Copyright 2011 Alibaba.com All right reserved. This software is the 
 * confidential and proprietary information of Alibaba.com ("Confidential 
 * Information"). You shall not disclose such Confidential Information and shall 
 * use it only in accordance with the terms of the license agreement you entered 
 * into with Alibaba.com. 
 */  
package com.alibaba.study.rpc.test;  
  
import com.alibaba.study.rpc.framework.RpcFramework;  
  
/** 
 * RpcConsumer 
 *  
 * @author william.liangf 
 */  
public class RpcConsumer {  
      
    public static void main(String[] args) throws Exception {  
        HelloService service = RpcFramework.refer(HelloService.class, "127.0.0.1", 1234);  
        for (int i = 0; i < Integer.MAX_VALUE; i ++) {  
            String hello = service.hello("World" + i);  
            System.out.println(hello);  
            Thread.sleep(1000);  
        }  
    }  
      
}
```
