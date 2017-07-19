# websocket 实现原理

首先,我们先在本机设置监听端口.

    localhost:9999

然后,新建html文件.在javascript部分写上

``` html
    <script type="text/javascript">
        let sock = new WebSocket("ws://127.0.0.1:9999"); //注意! 这里的地址协议是ws而不是http,代表使用websocket协议

        sock.onopen = function(){
            console.log(" WebSocket init ! ");
            sock.send("hello server")
        };

        sock.onclose = function(){
            console.log(" WebSocket close ! ");
        };

        sock.onmessage = function(message){
            console.log(message.data);
        };

        sock.onerror = function(error){
            for(attr in error){
                console.log(" WebSocket error :" + error + "\t\n");
            }
        };
    </script>
```

然后保存html文件并刷新一下浏览器,看看监听了什么内容?

### 握手处理

        GET / HTTP/1.1\r\n
        Host: 127.0.0.1:9999\r\n
        Connection: Upgrade\r\n
        Pragma: no-cache\r\n
        Cache-Control: no-cache\r\n
        Upgrade: websocket\r\n
        Origin: file://\r\n
        Sec-WebSocket-Version: 13\r\n
        User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64) \r\nAppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 \r\nSafari/537.36\r\n
        DNT: 1\r\n
        Accept-Encoding: gzip, deflate, br\r\n
        Accept-Language: zh-CN,zh;q=0.8\r\n
        Cookie: _ga=GA1.1.656631965.1457863228\r\n
        Sec-WebSocket-Key: gI83D50S5WxJc5ldFlnkwg==\r\n
        Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits\r\n\r\n

这个是前端请求后端使用websocket协议.一眼看下去可能跟普通的http协议没什么不同,但是主要看

    Connection:Upgrade

和

    Sec-WebSocket-Key: gI83D50S5WxJc5ldFlnkwg==

平时使用http协议进行连接的时候是 Connection:Keep-Alive,但是现在变成了 Connection:Upgrade.

    其实这里代表着请将http协议升级为websocket协议

服务器需要获取Sec-WebSocket-Key

然后服务器做出应答:

        HTTP/1.1 101 Switching Protocols\r\n
        Upgrade: websocket\r\n
        Connection: Upgrade\r\n
        Sec-WebSocket-Accept: 7jr5bZ4llRsMiyDqlPvBiWU4jY8=\r\n\r\n

这里服务器已经通过Upgrade: websocket对浏览器做出已经切换成websocket协议的回答

我们需要注意的是Sec-WebSocket-Accept,这个是将刚才获取到的Sec-WebSocket-Key跟一个固定的一串全局唯一的标识字符串（俗称魔串）258EAFA5-E914-47DA- 95CA-C5AB0DC85B11进行拼接,拼接过后的字符串需要使用 SHA-1 进行散列处理,接着将结果进行 base64 编码

ok,我们现在已经握手完毕

### 服务器接收消息

先祭出官方的消息帧协议图

        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
        +-+-+-+-+-------+-+-------------+-------------------------------+
        |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
        |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
        |N|V|V|V|       |S|             |   (if payload len==126/127)   |
        | |1|2|3|       |K|             |                               |
        +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
        |     Extended payload length continued, if payload len == 127  |
        + - - - - - - - - - - - - - - - +-------------------------------+
        |                               |Masking-key, if MASK set to 1  |
        +-------------------------------+-------------------------------+
        | Masking-key (continued)       |          Payload Data         |
        +-------------------------------- - - - - - - - - - - - - - - - +
        :                     Payload Data continued ...                :
        + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
        |                     Payload Data continued ...                |
        +---------------------------------------------------------------+

我通过查阅资料,知道了其实这里上面的0-7是一个字节(囧!为什么这里要标记8和9,感觉怪怪的)

这里主要有三个步骤:

1. [解析第一个字节](#first)
2. [解析第二个字节](#second)
3. [解码处理](#third)

<h4 id="first">解析第一个字节</h4>

首先看第一个0-7的字节内容.将这个字节跟 128(二进制 1000000)(十六进制 0x80)进行异或运算,得到的结果是 FIN 的值.这个值等于0代表还有下一帧,大于0代表结束.

再看那三个 RSV ,一般来说这三个 RSV 都为0,除非有协商过不为0.如果没有协商而不为0的话,那么这次连接就失败.

最后 opcode 这个是描述消息的类型,如果是一个未知的 opcode,也表示这次连接失败了.协议定义了以下的值:

                            这个字节的值为
    - %x0 表示连续的帧      128   (二进制 1000000)(十六进制 0x80)
    - %x1 表示 text 帧      129   (二进制 1000001)(十六进制 0x81)
    - %x2 表示二进制帧      130   (二进制 1000010)(十六进制 0x82)
    - %x3-7 预留给非控制帧  131   (二进制 1000011)(十六进制 0x83)
    - %x8 表示关闭连接帧    132   (二进制 1000100)(十六进制 0x84)
    - %x9 表示 ping         133   (二进制 1000101)(十六进制 0x85)
    - %xA 表示 pong         134   (二进制 1000110)(十六进制 0x86)
    - %xB-F 预留给控制帧    135   (二进制 1000111)(十六进制 0x87)

所以,这一步就跟128进行异或运算,得到的值如果为0,就等待下一帧.如果大于0,就判断是否在这一堆定义值中,如果不在,就连接失败,如果在,就得到了消息的类型.

<h4 id="second">解析第二个字节</h4>

接着我们来看第二个字节8-6.

MASK 表示是否有使用掩码对消息进行一个编码操作.1为使用,0为不使用.
Payload len 表示消息是否有用扩展字节标识消息的长度.有三个值:

    - 小于126 不使用扩展字节,并表示长度为Payload len
    - 等于126 使用接下来的两个扩展字节
    - 大于126 使用接下来的八个扩展字节

也就是说第二个字节首先要跟128来一个异或运算,得到 MASK 的值.接着跟127(二进制 011111111)(十六进制 0x7f)进行与运算,确定消息的长度

如果 MASK 等于1,读取后面四个字节

根据消息长度读取相应的消息

<h4 id='third'>解码处理</h4>

消息中的字节要根据位置与mask的字节进行解码(用python作为示例)

``` python
message = ''
i = 0
for d in data:
    message += chr(d ^ mask[i % 4])
    i += 1
```

## 服务器发送消息

第一步跟接收消息的第一步是一样的,也是要给第一个字节发送消息类型.

第二步就稍微有点不同的.一开始不需要设置 MASK 是否为1,因为浏览器不需要解码(至少在实践中是这样的),然后还是要给 Payload 按照解码那样设置消息的长度

第三部就是将消息转成字节,然后发送出去

## 参考资料

[WebSocket 实现原理](http://zeeyang.com/2017/07/02/websocket/)
