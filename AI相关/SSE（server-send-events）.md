
## 1、什么是SSE

严格地说，[HTTP 协议](https://www.ruanyifeng.com/blog/2016/08/http.html)无法做到服务器主动推送信息。但是，有一种变通方法，就是服务器向客户端声明，接下来要发送的是流信息（streaming）。

也就是说，发送的不是一次性的数据包，而是一个数据流，会连续不断地发送过来。这时，客户端不会关闭连接，会一直等着服务器发过来的新的数据流，视频播放就是这样的例子。本质上，这种通信就是以流信息的方式，完成一次用时很长的下载。

SSE 就是利用这种机制，使用流信息向浏览器推送信息。它基于 HTTP 协议，目前除了 IE/Edge，其他浏览器都支持。

## 2、SSE和WebSocket的区别

<table>
  <thead style="background-color: #f0f0f0">
    <tr>
      <th>特性</th>
      <th>Server-Sent Events (SSE)</th>
      <th>WebSocket</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong><font color='green'>通信方向</font></strong></td>
      <td>单向（服务器→客户端）</td>
      <td>双向（服务器↔客户端）</td>
    </tr>
    <tr>
      <td><strong><font color='green'>协议</font></strong></td>
      <td>基于 HTTP</td>
      <td>独立的 WebSocket 协议</td>
    </tr>
    <tr>
      <td><strong><font color='green'>连接建立</font></strong></td>
      <td>常规 HTTP 请求</td>
      <td>HTTP 升级请求</td>
    </tr>
    <tr>
      <td><strong><font color='green'>数据格式</font></strong></td>
      <td>文本（事件流格式）</td>
      <td>二进制或文本</td>
    </tr>
    <tr>
      <td><strong><font color='green'>自动重连</font></strong></td>
      <td>内置支持</td>
      <td>需要手动实现</td>
    </tr>
    <tr>
      <td><strong><font color='green'>浏览器支持</font></strong></td>
      <td>除 IE/Edge 外主流浏览器都支持</td>
      <td>所有现代浏览器</td>
    </tr>
    <tr>
      <td><strong><font color='green'>使用复杂度</font></strong></td>
      <td>简单</td>
      <td>相对复杂</td>
    </tr>
    <tr>
      <td><strong><font color='green'>适用场景</font></strong></td>
      <td>服务器向客户端推送通知、实时更新</td>
      <td>实时交互应用（如聊天、游戏）</td>
    </tr>
    <tr>
      <td><strong><font color='green'>消息分帧</font></strong></td>
      <td>不支持</td>
      <td>支持</td>
    </tr>
    <tr>
      <td><strong><font color='green'>CORS 限制</font></strong></td>
      <td>同源策略适用</td>
      <td>同源策略适用</td>
    </tr>
    <tr>
      <td><strong><font color='green'>连接数限制</font></strong></td>
      <td>每个浏览器标签6个HTTP连接（包括SSE）</td>
      <td>无硬性限制</td>
    </tr>
  </tbody>
</table>

## 3、客户端API

### 3.1 EventSource 对象

SSE 的客户端 API 部署在`EventSource`对象上。下面的代码可以检测浏览器是否支持 SSE。
``` javascript
 
 if ('EventSource' in window) {
   // ...
 }
 ```

使用 SSE 时，浏览器首先生成一个`EventSource`实例，向服务器发起连接。

```javascript
 
 var source = new EventSource(url);
  ```

上面的`url`可以与当前网址同域，也可以跨域。跨域时，可以指定第二个参数，打开`withCredentials`属性，表示是否一起发送 Cookie。

 ```javascript
 
 var source = new EventSource(url, { withCredentials: true });
 ```

`EventSource`实例的`readyState`属性，表明连接的当前状态。该属性只读，可以取以下值。

- 0：相当于常量`EventSource.CONNECTING`，表示连接还未建立，或者断线正在重连。
- 1：相当于常量`EventSource.OPEN`，表示连接已经建立，可以接受数据。
- 2：相当于常量`EventSource.CLOSED`，表示连接已断，且不会重连。

### 3.2 基本用法

连接一旦建立，就会触发`open`事件，可以在`onopen`属性定义回调函数。

```javascript

 source.onopen = function (event) {
   // ...
 };
 
 // 另一种写法
 source.addEventListener('open', function (event) {
   // ...
 }, false);
 ```

客户端收到服务器发来的数据，就会触发`message`事件，可以在`onmessage`属性的回调函数。

```javascript
 
 source.onmessage = function (event) {
   var data = event.data;
   // handle message
 };
 
 // 另一种写法
 source.addEventListener('message', function (event) {
   var data = event.data;
   // handle message
 }, false);
 ```

上面代码中，事件对象的`data`属性就是服务器端传回的数据（文本格式）。

如果发生通信错误（比如连接中断），就会触发`error`事件，可以在`onerror`属性定义回调函数。

```javascript
 
 source.onerror = function (event) {
   // handle error event
 };
 
 // 另一种写法
 source.addEventListener('error', function (event) {
   // handle error event
 }, false);
 ```

`close`方法用于关闭 SSE 连接。

```javascript
 
 source.close();
 ```

### 3.3 自定义事件

默认情况下，服务器发来的数据，总是触发浏览器`EventSource`实例的`message`事件。开发者还可以自定义 SSE 事件，这种情况下，发送回来的数据不会触发`message`事件。

```javascript
 
 source.addEventListener('foo', function (event) {
   var data = event.data;
   // handle message
 }, false);
 ```

上面代码中，浏览器对 SSE 的`foo`事件进行监听。如何实现服务器发送`foo`事件，请看下文。


##  4、服务器实现

### 4.1 数据格式

服务器向浏览器发送的 SSE 数据，必须是 UTF-8 编码的文本，具有如下的 HTTP 头信息。

```markup
 
 Content-Type: text/event-stream
 Cache-Control: no-cache
 Connection: keep-alive
 ```

上面三行之中，第一行的`Content-Type`必须指定 MIME 类型为`event-steam`。

每一次发送的信息，由若干个`message`组成，每个`message`之间用`\n\n`分隔。每个`message`内部由若干行组成，每一行都是如下格式。

```markup
 
 [field]: value\n
 ```

上面的`field`可以取四个值。

- data
- event
- id
- retry

此外，还可以有冒号开头的行，表示注释。<font color='red'>通常，服务器每隔一段时间就会向浏览器发送一个注释，保持连接不中断。</font>

 ```markup

 : This is a comment
 ```

下面是一个例子。

```markup
 
 : this is a test stream\n\n
 
 data: some text\n\n
 
 data: another message\n
 data: with two lines \n\n
 ```

### 4.2 data 字段

数据内容用`data`字段表示。

 ```markup
 
 data:  message\n\n
 ```

如果数据很长，可以分成多行，最后一行用`\n\n`结尾，前面行都用`\n`结尾。

```markup
 
 data: begin message\n
 data: continue message\n\n
 ```

下面是一个发送 JSON 数据的例子。

```markup
 
 data: {\n
 data: "foo": "bar",\n
 data: "baz", 555\n
 data: }\n\n
 ```

### 4.3 id 字段

数据标识符用`id`字段表示，相当于每一条数据的编号。

```markup
 
 id: msg1\n
 data: message\n\n
 ```

浏览器用`lastEventId`属性读取这个值。一旦连接断线，浏览器会发送一个 HTTP 头，里面包含一个特殊的`Last-Event-ID`头信息，将这个值发送回来，用来帮助服务器端重建连接。因此，这个头信息可以被视为一种同步机制。

### 4.4 event 字段

`event`字段表示自定义的事件类型，默认是`message`事件。浏览器可以用`addEventListener()`监听该事件。

```markup
 
 event: foo\n
 data: a foo event\n\n
 
 data: an unnamed event\n\n
 
 event: bar\n
 data: a bar event\n\n
 ```

上面的代码创造了三条信息。第一条的名字是`foo`，触发浏览器的`foo`事件；第二条未取名，表示默认类型，触发浏览器的`message`事件；第三条是`bar`，触发浏览器的`bar`事件。

下面是另一个例子。

```markup
 
 event: userconnect
 data: {"username": "bobby", "time": "02:33:48"}
 
 event: usermessage
 data: {"username": "bobby", "time": "02:34:11", "text": "Hi everyone."}
 
 event: userdisconnect
 data: {"username": "bobby", "time": "02:34:23"}
 
 event: usermessage
 data: {"username": "sean", "time": "02:34:36", "text": "Bye, bobby."}
 ```

### 4.5 retry 字段

服务器可以用`retry`字段，指定浏览器重新发起连接的时间间隔。

```markup
 
 retry: 10000\n
 ```

两种情况会导致浏览器重新发起连接：一种是时间间隔到期，二是由于网络错误等原因，导致连接出错。

```JAVA
/**  
 * Sends an SSE event to a client. * @param writer The writer to send the event through  
 * @param eventType The type of event (message or endpoint)  
 * @param data The event data  
 * @throws IOException If an error occurs while writing the event  
 */private void sendEvent(PrintWriter writer, String eventType, String data) throws IOException {  

   // java版本，服务器向客户端发送消息
   writer.write("event: " + eventType + "\n");   // message 、 endpoint
   writer.write("data: " + data + "\n\n");  
   writer.flush();  
  
   if (writer.checkError()) {  
      throw new IOException("Client disconnected");  
   }  
}
```

**引用**：

https://developer.mozilla.org/zh-CN/docs/Web/API/Server-sent_events/Using_server-sent_events?app_lang=zh-CN

https://www.ruanyifeng.com/blog/2017/05/server-sent_events.html
