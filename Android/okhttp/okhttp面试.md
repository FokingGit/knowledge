#### socket和WebSocket的差别 websocket和http长链接的差别

1. socket是一个抽象的接口,用于链接通信两端.

   websocket是一个应用层的协议,是全双工的,受限通过http协议建立通信来建立链接,此时的http链接中会添加一个请求头Upgrade：websocket

2. http协议和websocket协议的区别

   - 相同点
     - 都是基于TCP链接
     - 都是应用层的协议

   - 不同点
     - webSocket协议是双向通信协议,也就是全双工的
     - webSocket的建立需要通过握手

   - 二者之间的联系
     - websocket的建立需要通过http