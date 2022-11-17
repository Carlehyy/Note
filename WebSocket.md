# 整体介绍
本案例基于 `Vue2` +  `SpringBoot`，旨在掌握 `websocket` 基本操作，实现的功能有：① 前后端的单播和广播功能。② 配置合理的、规范的通信格式。

## 一、后端代码

### pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.71</version>
</dependency>
```

### MessageBO.java

> 本文件规定客户端发送至服务端的数据格式。
>
> - isBroadcast: 单播或广播
> - target: 单播对象
> - content: 通信内容

```java
@Data
public class MessageBO {
    private Boolean isBroadcast;
    private String target;
    private String content;
}
```
### MessageVO.java

> 本文件规定服务端发送至客户端的数据格式。
>
> - isSystem: 单播或广播
> - content: 通信内容

```java
@Data
public class MessageVO {

    private Boolean isSystem;
    private String content;

}
```
### MessageDecoder.java

> 本文件旨在将客户端发送至服务端的字符串解析成对象格式。
>
> 注意： `willDecode` 函数返回值表示是否需要解析，设置成 `false` 则 `decode` 函数不执行，此处设置成 `true`。

```java
public class MessageDecoder implements Decoder.Text<MessageBO> {
    @Override
    public MessageBO decode(String msg) throws DecodeException {
        return JSON.parseObject(msg, MessageBO.class);
    }
    @Override
    public boolean willDecode(String s) {return true;}
    @Override
    public void init(EndpointConfig endpointConfig) {}
    @Override
    public void destroy() {}
}
```
### MessageEncoder.java

> 本文件旨在将服务端发送至客户端的对象封装成字符串格式。

```java
public class MessageEncoder implements Encoder.Text<MessageVO> {
    @Override
    public String encode(MessageVO messageVO) throws EncodeException {
        return JSON.toJSONString(messageVO);
    }
    @Override
    public void init(EndpointConfig endpointConfig) {}
    @Override
    public void destroy() {}
}
```
### WebSocketConfig.java

> 通过配置文件，让 `Springboot` 自动扫描 `ServerEndpoint` 文件并注入到 `Bean` 容器。

```java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```
### Websocket.java

> websocket功能文件，通过注解 `ServerEndpoint` 监听指定路径，并设置解码器和编码器格式化通信数据。

```java
@Component
@ServerEndpoint(value = "/websocket", decoders = MessageDecoder.class, encoders = MessageEncoder.class)
public class Websocket {

    private static HashMap<Integer, Websocket> onlineUsers = new HashMap<>();
    private Integer id;
    private Session session;

    @OnOpen
    public void onOpen(Session session, EndpointConfig config) {
        this.session = session;
        this.id = onlineUsers.size()+1;
        onlineUsers.put(this.id, this);
        broadcast("新用户登录");
    }

    @OnClose
    public void onClose(Session session, CloseReason closeReason) {
        onlineUsers.remove(this.id);
    }

    @OnMessage
    public void onMessage(Session session, MessageBO messageBO) {
        System.out.println(messageBO);
        // 判断消息是群发还是指定发送
        if (messageBO.getIsBroadcast()) {
            broadcast(messageBO.getContent());
        } else {
            unicast(messageBO.getContent(), JSON.parseObject(messageBO.getTarget(), Integer.class));
        }
    }

    @OnError
    public void onError(Session session, Throwable throwable) {
        System.out.println(onlineUsers);
        System.out.println("websocket连接发生错误.");
        throwable.printStackTrace();
    }

    // 单播静态方法
    public static void unicast(String msg, Integer target) {
        try {
            Websocket websocket = onlineUsers.get(target);
            MessageVO messageVO = new MessageVO();
            messageVO.setIsSystem(false);
            messageVO.setContent(msg);
            websocket.session.getBasicRemote().sendObject(messageVO);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (EncodeException e) {
            e.printStackTrace();
        } finally {
        	System.out.println("websocket服务端《单播》消息异常");
        }
    }
    // 广播静态方法
    public static void broadcast(String msg) {
        try {
            Set<Integer> ids = onlineUsers.keySet();
            for (Integer id : ids) {
                Websocket websocket = onlineUsers.get(id);
                MessageVO messageVO = new MessageVO();
                messageVO.setIsSystem(true);
                messageVO.setContent(msg);
                websocket.session.getBasicRemote().sendObject(messageVO);
            }
        } catch (Exception e) {
            System.out.println("websocket服务端《广播》消息异常");
            e.printStackTrace();
        }
    }

}
```
## 二、前端代码

> 客户端发送或者接收 `websocket` 的数据格式均为字符串，所以前端需在发送时进行字符串封装，在接收时进行字符串解析。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>客户端-服务器双向通信</title>
    <!-- 引入vue.js -->
    <script src="https://cdn.jsdelivr.net/npm/vue@2.6.10/dist/vue.js"></script>
	<!-- axios.js -->
	<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
    <!-- 引入element-ui样式 -->
    <link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css" />
    <!-- 引入element-ui组件库 -->
    <script src="https://unpkg.com/element-ui/lib/index.js"></script>
    <style>
      #app {
        /* width: 1200px; */
        height: 870px;
        margin: 0;
        padding: 20px 50px 20px;
        display: flex;
        flex-direction: row;
        flex-wrap: wrap;
        justify-content: space-around;
        align-items: center;
        background-color: #ccc;
      }
      .flexBox-1 {
        width: 58%;
        height: 100%;
      }
      .flexBox-2 {
        width: 36%;
        height: 100%;
      }
	  .flexBox-3 {
        width: 50%;
        /* height: 100%; */
		margin: 50px auto 20px;
      }
    </style>
  </head>
  <body>
    <div id="app">
		<el-card v-if="isLogin" class="flexBox-1" shadow="hover">
	        <h1 align="right">
	          <h4 style="display: inline-block;margin-right: 12px;">当前用户：{{currentUser.username}}</h4>
	          <el-button type="text" @click="logout">退出</el-button>
	        </h1>
	        <el-button type="primary" @click="beginConnection">开启连接</el-button>
	        <el-button type="warning" @click="closeConnection">关闭连接</el-button>
	        <el-divider></el-divider>
	        <p>文字略...</p>
	        <el-form ref="form" :model="message" label-width="80px">
	          <el-form-item label="消息输入">
	            <el-input type="textarea" v-model="message.content"></el-input>
	          </el-form-item>
	          <el-form-item label="">
	            <el-button type="success" @click="broadcastMessage">广播消息</el-button>
	          </el-form-item>
	        </el-form>
		</el-card>
		<el-card v-if="isLogin" class="flexBox-2" shadow="hover">
			<h1 align="center">系统广播</h1>
			<el-divider></el-divider>
	        <div>
	          <p v-for="(msg, i) in globalMessage" :key="i">{{msg}}</p>
	        </div>
		</el-card>
		<el-card v-else class="flexBox-3" shadow="hover">
			<h1 align="center">ONLINE CHAT</h1>
			<el-form :model="user" label-width="80px">
				<el-form-item label="username">
					<el-input v-model="user.username"></el-input>
				</el-form-item>
				<el-form-item label="password">
					<el-input v-model="user.password"></el-input>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" @click="login">login</el-button>
				</el-form-item>
			</el-form>
		</el-card>
    </div>
    <script>
      const app = new Vue({
        el: "#app",
        data: {
			isLogin: false,
			user: {
				username: null,
				password: null
			},
          	message: {
	            content: null,
	            target: null,
          	},
			currentUser: null,
			globalMessage: [],
          	websocket: null,
		},
        methods: {
          beginConnection() {
            this.initWebsocket()
          },
          closeConnection() {
            this.websocket.close()
          },
          broadcastMessage() {
            // 注意要使用JSON进行字符串封装
            var msg = JSON.stringify({
              isBroadcast: true,
              target: null,
              content: this.message.content
            })
            this.websocket.send(msg)
          },
          // 创建websocket通信
          initWebsocket() {
            // wsurl格式: ws://ip:port/address
            const wsurl = "ws://localhost:8080/websocket"
            this.websocket = new WebSocket(wsurl)
            this.websocket.onopen = this.websocketopen
            this.websocket.onclose = this.websocketclose
            this.websocket.onmessage = this.websocketmessage
            this.websocket.onerror = this.websocketerror
          },
          websocketopen() {
            this.$message.success("开启连接")
          },
          // 通信关闭
          websocketclose(event) {
            this.$message.success("关闭连接")
            console.log(event)
          },
          // 接收服务端消息
          websocketmessage(event) {
            // 注意要使用JSON进行字符串解析
            var msg = JSON.parse(event.data)
            if (msg.isSystem) {
              this.$message.warning(`您有一条新的广播消息, 请查收`)
              this.globalMessage.push(msg.content)
            } else {
              this.$message.warning(`点播消息: ${msg.content}`)
            }
          },
          // 通信异常
          websocketerror(event) {
            this.$message.error("连接出错")
            console.log(event)
          },
		  // 登录
		  login() {
            if(this.user.username && this.user.password) {
              axios.post("http://localhost:8080/user/login", this.user).then(response => {
                this.$message.success("登录成功")
                console.log(response)
                this.isLogin = true
                this.currentUser = JSON.parse(JSON.stringify(this.user))
              }).catch(error => {
                console.log(error)
              })
            } else {
               this.$message.warning("用户名或密码不能为空, 请重新检查.")
            }
		  },
          // 注销
          logout() {
            axios.post("http://localhost:8080/user/logout")
            .then(response => {
              this.$message.success("您已退出")
              console.log(response)
              this.isLogin = false
              this.currentUser = null
              Object.keys(this.user).forEach(key => this.user[key] = null)})
            .catch(error => {
			  console.log(error)
			 })
          },
        },
      });
    </script>
  </body>
</html>
```

## 三、WebSocket在线测试工具

http://wstool.js.org/

http://www.websocket-test.com/

http://coolaf.com/tool/chattest