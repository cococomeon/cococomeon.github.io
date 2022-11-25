
```java
java.net.Socket socket = new java.net.Socket("host", port);
```


```java
public class Socket implements java.io.Closeable {
    public Socket(String host, int port) throws UnknownHostException, IOException {
        this(
            host != null ? new InetSocketAddress(host, port) : new InetSocketAddress(InetAddress.getByName(null), port),
            (SocketAddress) null, true);
    }



}


public class InetSocketAddress
  extends SocketAddress {

    public InetSocketAddress(String hostname, int port) {
        checkHost(hostname);
        InetAddress addr = null;
        String host = null;
        try {
            addr = InetAddress.getByName(hostname);//解析hostname DNS，这个动作在new socket的时候就做了
        } catch (UnknownHostException e) {
            host = hostname;
        }
        holder = new InetSocketAddressHolder(host, addr, checkPort(port));
    }

}

public class InetAddress implements java.io.Serializable {
    public static InetAddress getByName(String host) throws UnknownHostException {
        return InetAddress.getAllByName(host)[0];
    }

  public static InetAddress[] getAllByName(String host)
    throws UnknownHostException {
    return getAllByName(host, null);
  }
}





```

DNS不仅产生于操作系统，浏览器，应用程序以及IPS运营服务商都会对DNS进行缓存。





- [浅析 DNS 缓存技术及应用考虑](https://www.infoq.cn/article/8qmbxvabn3ec8vt5itxw)
