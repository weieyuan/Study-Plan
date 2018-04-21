## 正向代理和反向代理
**#正向代理**  
正向代理位于客户端和原始服务器端之间的服务器，为了从原始服务器中取到内容，客户端向代理服务器发送请求并且告知真正请求数据的目标服务器，代理服务器向目标服务器转发请求并获取数据，然后返回数据给客户端。

作用：  
1. 访问无法访问的资源
2. 加速访问，通过不同的路径去访问资源
3. 缓存作用
4. 权限控制
5. 隐藏访问者，目标服务器只能获取代理服务器的信息

**#反向代理**  
客户端不感知代理的存在，直接向代理服务器请求数据，代理服务器将请求转发到目标服务器获取数据并返回给客户端。

作用：  
1. 隐藏原始服务器，让客户端认为代理服务器就是原始服务器
2. 缓存作用