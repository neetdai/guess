# websocket的协议
首先,我们先在本机设置监听端口.

    localhost:9999

然后,新建html文件.在javascript部分写上

    ````
    <script type="text/javascript">
        let sock = new WebSocket("ws://127.0.0.1:9999");

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
    ````

然后保存html文件并刷新一下浏览器,看看监听了什么?
