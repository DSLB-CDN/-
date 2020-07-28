上周和老师开会讨论的时候，老师提到了一个问题：当四层负载均衡和七层负载均衡同时运作的时候，对于客户端来说，服务器由于反向代理的特性，会对客户端屏蔽真实的web服务器IP地址，客户端只需要知道代理服务器的地址就可以进行访问。

对于web服务器来说，如果不进行处理，客户端的源IP地址同样会被屏蔽，这时web服务器会将所有访问都看成是代理服务器发出的，这样对于web服务器中的一些需要进行签名验证，客户端IP限定等的情况就会有影响。

部署的后端web项目  **ruoyi** 里有一个选项是显示在线用户信息，其中有一栏是显示在线用户的IP地址。下面进行测试：

![mark](http://image.gwbiubiubiu.com/blog/20200722/zUqYYAy55Jck.png)

### 访问后端服务器

后端服务器的端口号为5058和5062，直接访问的结果如下图

![mark](http://image.gwbiubiubiu.com/blog/20200722/0QHaxEcbC8NG.png)

其中120.239.204.145是客户端IP地址，说明直接访问后端服务器，是可以成功获取IP地址的。

### 访问七层Nginx服务器

七层Nginx服务器的端口号为5052、5054、以及5056，这里用于做域名解析以及负载均衡，访问5052端口的结果如图：

![mark](http://image.gwbiubiubiu.com/blog/20200722/J4FmUGq12ndI.png)

可以看出访问七层Nginx服务器的时候，后端也是可以获取客户端的源IP地址的。

### 访问四层Nginx服务器

四层的Nginx服务器作用在TCP层，访问的端口号为5050，

![mark](http://image.gwbiubiubiu.com/blog/20200722/J4FmUGq12ndI.png)

访问四层Nginx服务器的是时候，可以看到后端显示的主机地址是10.2.1.226，该地址为内网地址，说明此时外网地址已经被屏蔽了。

### 后端代码分析

用IDEA打开后端项目，查看此部分获取IP地址的代码，如下所示。

```java
public static String getIpAddr(HttpServletRequest request)
{
    if (request == null)
    {
        return "unknown";
    }
    String ip = request.getHeader("x-forwarded-for");
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
    {
        ip = request.getHeader("Proxy-Client-IP");
    }
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
    {
        ip = request.getHeader("X-Forwarded-For");
    }
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
    {
        ip = request.getHeader("WL-Proxy-Client-IP");
    }
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
    {
        ip = request.getHeader("X-Real-IP");
    }

    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
    {
        ip = request.getRemoteAddr();
    }
    return "0:0:0:0:0:0:0:1".equals(ip) ? "127.0.0.1" : EscapeUtil.clean(ip);
}
```

其中 HttpServletRequest对象代表客户端的请求，当客户端通过HTTP协议访问服务器时，HTTP请求头中的所有信息都封装在这个对象中，通过这个对象提供的方法，可以获得客户端请求的所有信息。 

比如用getRemoteAddr()，返回客户端的IP地址，但是此方法只适用于客户端直接访问服务器的情况，对于中间有代理服务器的时候，用getRemoteAddr()方法得到的IP地址就是最后直接连到web服务器的代理服务器的地址，所以一般不会直接使用。

那么如何在现有的代理模式下获取正确的客户端IP地址？

根据代码我们可以得知，首先后端程序调用的是getHeader()，也就是读取客户端HTTP请求头中的某些字段，相关请求头的解释 如下

> - X-Forwarded-For ：这是一个 Squid 开发的字段，只有在通过了HTTP代理或者负载均衡服务器时才会添加该项。　　　　
>
> 格式为X-Forwarded-For:client1,proxy1,proxy2，一般情况下，第一个IP为客户端真实IP，后面的为经过的代理服务器IP。现在大部分的代理都会加上这个请求头。
>
> - Proxy-Client-IP/WL- Proxy-Client-IP ：这个一般是经过apache http服务器的请求才会有，用apache http做代理时一般会加上Proxy-Client-IP请求头，而WL-Proxy-Client-IP是他的weblogic插件加上的头。
> - HTTP_CLIENT_IP ：有些代理服务器会加上此请求头。
> - X-Real-IP ：nginx代理一般会加上此请求头。

查看上层nginx转发配置如下：

```shell
location /prod-api/{
	proxy_set_header Host $http_host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header REMOTE-HOST $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_pass http://10.2.1.231:8080/;
}
```

可以看出$remote_addr为客户端IP地址，再Nginx转发的时候将$remote_addr放进了HTTP的“X-Real-IP”字段和“REMOTE-HOST”字段。

$proxy_add_x_forwarded_for变量包含客户端请求头中的"X-Forwarded-For"，与$remote_addr用逗号分开，如果没有"X-Forwarded-For" 请求头，则$proxy_add_x_forwarded_for等于$remote_addr。

> 当Nginx设置X-Forwarded-For于$proxy_add_x_forwarded_for后会有两种情况发生：
>
> 1、如果从CDN过来的请求没有设置X-Forwarded-For头（通常这种事情不会发生），而到了我们这里Nginx设置将其设置为$proxy_add_x_forwarded_for的话，X-Forwarded-For的信息应该为CDN的IP，因为相对于Nginx负载均衡来说客户端即为CDN，这样的话，后端的web程序时死活也获得不了真实用户的IP的。
>
> 2、CDN设置了X-Forwarded-For，我们这里又设置了一次，且值为$proxy_add_x_forwarded_for的话，那么X-Forwarded-For的内容变成 ”客户端IP,Nginx负载均衡服务器IP“如果是这种情况的话，那后端的程序通过X-Forwarded-For获得客户端IP，则取逗号分隔的第一项即可。

### DEMO测试

将后端获取源IP的流程分析清楚之后，写了个Demo简单测试一下在不同的端口进入的时候各个字段的结果。部分代码如下

```java
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write(getIpAddr(req) + "\n");
    		//后端系统处理过的结果
        resp.getWriter().write(req.getRemoteAddr() + "\n");
        	//直接读取客户端的IP地址
        resp.getWriter().write("x-forwarded-for: "+req.getHeader("x-forwarded-for") +"\n"); 	//显示x-forwarded-for字段
        resp.getWriter().write("Proxy-Client-IP: "+req.getHeader("Proxy-Client-IP")+"\n");//显示Proxy-Client-IP字段
        resp.getWriter().write("X-Forwarded-For: " + req.getHeader("X-Forwarded-For")+"\n");//显示X-Forwarded-For字段
        resp.getWriter().write("WL-Proxy-Client-IP: " + req.getHeader("WL-Proxy-Client-IP")+"\n");//显示WL-Proxy-Client-IP字段
        resp.getWriter().write("X-Real-IP: "+ req.getHeader("X-Real-IP")+"\n");
    	//显示X-Real-IP字段
    }
```

打包部署后，进行测试，查看结果：

访问后端服务器

![mark](http://image.gwbiubiubiu.com/blog/20200723/nfWxWK21GHq9.png)

访问七层Nginx服务器

![mark](http://image.gwbiubiubiu.com/blog/20200723/NJC9RzQWt4ph.png)

访问四层Nginx服务器

 ![mark](http://image.gwbiubiubiu.com/blog/20200723/Ha7tY7FVKLlW.png)

### 测试结果分析

可以看到，客户端直接访问web服务器的时候，HTTP head的字段都为null，此时后端通过getRemoteAddr() 方法可以获取到正确的客户端IP地址；

当从七层Nginx服务器端口访问的时候，就无法通过getRemoteAddr() 方法直接获取到客户端IP地址了，此时是通过读取X-Forwarded-For字段来读到正确的IP地址的；

当通过四层Nginx服务器端口访问的时候，也是通过读取X-Forwarded-For字段来获取IP地址，可是在经过四层负载均衡的时候，已经将源IP地址丢弃了，所以到七层的时候误将四层nginx服务器的地址当作了源IP地址，才会导致真实的客户端IP地址获取不到。

### 后续改进方案

为了在四层将源IP地址保存住，需要在四层负载均衡的时候把真正的源地址，插入在TCP-option里，这时只要在7层设备上装toa模块，就可以实现hook进7层设备的源地址获取过程，让其取源地址时从TCP-option去获取，并放进http的X-forwarded-for字段，这样就可以在后端获取到源地址了。