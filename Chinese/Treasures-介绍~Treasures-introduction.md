#Treasures介绍
##描述
[Treasures](https://github.com/NetEase/treasures) 游戏是从 [LordOfPomelo](https://github.com/NetEase/lordofpomelo) 中抽取出来的, 去掉了大量的游戏逻辑, 用以更好的展示 [Pomelo](https://github.com/NetEase/pomelo) 框架的用法以及运作机制. 

Treasures 很简单, 输入一个用户名后, 会随机得到一个游戏角色并进入游戏场景. 在游戏场景中地上会散落一些宝物, 每个宝物都有分数, 玩家操作游戏角色去捡起地上的宝物, 然后得到相应的分数. 

##安装和运行
安装 `pomelo`

```bash
npm install -g pomelo
```
获取源码
```bash
git clone https://github.com/NetEase/treasures.git
```
安装 `npm` 依赖包（先进入项目目录）
```bash
sh npm-install.sh
```
启动 `web-server`  (先进入`web-server`目录)
```bash
node app.js
```
启动 `game-server` (先进入`game-server`目录)
```bash
pomelo start
```
在浏览器中访问 [http://localhost:3001](http://localhost:3001) 进入游戏.

如有问题, 可以参照 [pomelo快速使用指南](https://github.com/NetEase/pomelo/wiki/pomelo%E5%BF%AB%E9%80%9F%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97)

也可以参照 [tutorial1 分布式聊天](https://github.com/NetEase/pomelo/wiki/tutorial1--%E5%88%86%E5%B8%83%E5%BC%8F%E8%81%8A%E5%A4%A9)

##架构
Treasures 分为 web-Server 和 game-Server 两部分. 

* `web-server` 是用 Express 建立的最一个基础的 http 服务器, 用来为浏览器页面的访问提供服务. 

* `game-server` 是 WebSocket 服务器, 用来运行整个游戏的逻辑. 

首先, 通过配置文件来看 `game-server` 的具体架构 `game-server/config/server.json`
```javascript
{
  "development": {
    "connector": [
      {"id": "connector-server-1", "host": "127.0.0.1", "port": 3150, "clientPort": 3010, "frontend": true},
      {"id": "connector-server-2", "host": "127.0.0.1", "port": 3151, "clientPort": 3011, "frontend": true}
    ],
    "area": [
      {"id": "area-server-1", "host": "127.0.0.1", "port": 3250, "areaId": 1}
    ],
    "gate": [
      {"id": "gate-server-1", "host": "127.0.0.1", "clientPort": 3014, "frontend": true}
    ]
  }
}
```
可以看到, 服务端是由以下几个部分构成：

* 1 个 `gate` 服务器, 主要用于负载均衡, 把来自客户端的连接分散到两个 `connector` 服务器上. 
* 2 个 `connector` 服务器, 主要用于接收和发送消息. 
* 1 个 `area` 服务器, 主要用于驱动游戏场景和游戏逻辑

服务器之间的关系, 如下图所示：

![treasure-arch](http://pomelo.netease.com/resource/documentImage/treasure-arch.png)

##源码分析
#####通过游戏流程来分析代码:

###1. 连接服务器
客户端 `web-server/public/js/main.js` 的 `entry` 方法中:

```javascript
pomelo.request('gate.gateHandler.queryEntry', {uid: name}, function(data) {
  //...
});
```

服务端 `game-server/app/servers/gate/handler/gateHandler.js` 的 `queryEntry`方法中:

```javascript
Handler.prototype.queryEntry = function(msg, session, next) {
  // ...
  // 通过crc算法将玩家输入的用户名字符串转换为整数, 然后对connector服务器个数取余hash到某个connector服务器上.
  // 最后将要连接的 connector 服务器的 host 和 port 返回给客户端.
  next(null, {code: Code.OK, host: res.host, port: res.wsPort});
};
```

这样客户端就能连接到所分配的 `connector` 服务器上了. 

###2. 进入游戏
在与 `connector` 服务器建立连接之后, 就可以进入游戏了:

```javascript
pomelo.request('connector.entryHandler.entry', {name: name}, function(data) {
  // ...
});
```

在客户端第一次向 `connector` 服务器发出请求时, 服务器会将 `session` 信息进行初始化和绑定:

```javascript
session.bind(playerId); // session 与 playerId 绑定
session.set('areaId', 1); // 设置玩家 areaId
```
并将该 `connector` 服务器的序号与一个自增的变量 `id` 组合生成该角色的 `playerId`, 最后将生成的这个 `playerId` 发送给客户端.

客户端向服务端发起进入场景的请求：

```javascript
pomelo.request("area.playerHandler.enterScene", {name: name, playerId: data.playerId}, function(data) {
  // ...
});
```

客户端向服务端发送请求后, 请求先到达 `connector` 服务器, 然后 `connector` 服务器根据pomelo框架的默认转发规则(`pomelo/lib/components/proxy.js`中的`defaultRoute`函数), 将请求路由到相应的 `area` 服务器(本例子中只有一个`area`服务器), `area` 服务器中的 `playerHandler` 再处理相应的请求. 这样玩家就加入到游戏场景中了. 

在一个玩家加入到游戏场景之后, 其他玩家必须能即时的看到这个玩家的加入, 所以服务端必须将消息广播到在此游戏场景中的所有玩家. 
建立 `channel`, 所有加入此游戏场景的玩家都会加入到这个 `channel` 中

```javascript
// game-server/app/models/area.js 的 addEntity 函数中
// 获取 channel, 如果没有就创建一个, 然后将玩家加入 channel
getChannel().add(e.id, e.serverId);
```

当 `area` 中有玩家加入, 或其他事件发生改变时, 这些信息都会被推送到在这个 `channel` 中的每个玩家. 例如有玩家加入时：

```javascript
// game-server/app/models/area.js 的 entityUpdate 函数中
getChannel().pushMessage({route: 'addEntities', entities: added});
// 注: entityUpdate 是由 game-server/app/models/timer.js 中的 tick 函数调用的(详见下面介绍).
```

这些消息都是通过 `connector` 服务器发送到客户端的. `area` 中的消息是通过`session.frontendId` 来决定是由哪个 `connector` 服务器发出去. 

客户端接受消息:
```javascript
// web-server/public/js/msgHandler.js 中的 init 函数:
// 当有新玩家加入时, 服务端会广播消息给所有玩家. 客户端通过这个路由绑定, 来获取消息.
pomelo.on('addEntities', function(data) {
  // ...
});
```

###3. Area 服务器
`area` 服务器是一个由 `tick` 来驱动的游戏场景. 每个tick(100ms)都会对场景中 `entity` 的状态进行更新. 如果 `entities` 的状态发生了改变, 那么新的状态会被推送到所有相关客户端.
```javascript
function tick() {
  //run all the action
  area.actionManager().update();
  // update entities
  area.entityUpdate();
  // update rank
  area.rankUpdate();
}
```

例如玩家发起一个 `move` 动作:

客户端
```javascript
// 向服务端发送 move 请求request:
pomelo.request('area.playerHandler.move', {targetPos: targetPos}, function(result) {...});
```
服务端 `playerHandler` 接受请求：
```javascript
handler.move = function(msg, session, next) {
  // ...
  // 产生一个 move action
  var action = new Move({
    entity: player,
    endPos: endPos
  });
  // ...
  // 并将该 action 加入到 actionManager 中:
  if (area.timer().addAction(action)) { ... });
  // ...
});
```
然后这个 `action` 会在下个 `tick` 中更新. 

###4. 客户端发送和接受消息
客户端和服务端的通讯有以下几种方式：

* Request - Response 方式

```javascript
// 向 connector 发送请求, 参数 {name: name}
pomelo.request('connector.entryHandler.entry', {name: name}, function(data) {
  // 回调函数得到请求的返回结果
  // do something
});
```

* Notify (向服务端发送通知)

```javascript
// 向服务端发送notify通知
pomelo.notify("***.***.***", params);
```

* Push （服务端主动发送消息到客户端）

```javascript
// 当有新玩家加入时, 服务端会广播消息给所有玩家. 客户端通过这个路由绑定, 来获取消息:
pomelo.on('addEntities', function(data) {
  // ...
});
```

###5. 离开游戏

当玩家离开游戏时, `connector` 服务器会先收到断开的消息. 然后需要在 `area` 服务器中将用户剔除, 并广播消息给其他在线玩家.
 
因为服务器之间的进程都是独立的, 所以这就涉及到一个 `RPC` 调用. 好在 Pomelo 框架对 `RPC` 做了很好的封装, 例如:
`area` 服务器想要提供一系列的 `remote` 接口供其他服务器进程调用, 只需要在 `servers/area` 目录下创建一个 `remote` 目录, 在 `remote` 目录下的文件暴露出来的接口, 都可以作为 `RPC` 调用接口. 

例如, 玩家离开:

```javascript
// connector 中对 session 绑定事件, 当 session 关闭时, 触发事件
session.on('closed', onUserLeave.bind(null, self.app));
var onUserLeave = function (app, session, reason) {
  if (session && session.uid) {
    // rpc 调用
    app.rpc.area.playerRemote.playerLeave(session, {playerId: session.get('playerId'), areaId: session.get('areaId')}, null);
  }
};
```

对应的 `area/remote/playerRemote.js` 中 `playerLeave` 方法

```javascript
exp.playerLeave = function(args, cb) {
  // ...
  // 发出通知
  area.getChannel().pushMessage({route: 'onUserLeave', code: consts.MESSAGE.RES, playerId: playerId});
  // ...
};
```
这样就轻松的完成了一个跨进程的调用.


$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

YouDao:

# Treasures to introduce 
# # description 
[Treasures] (https://github.com/NetEase/treasures), the game is extracted from the [LordOfPomelo] (https://github.com/NetEase/lordofpomelo), get rid of a lot of game logic, in order to better display [Pomelo] (https://github.com/NetEase/pomelo) the use of the framework and operation mechanism. 
 
Treasures is very simple, enter a user name, will get a random game characters and into the game scene. In the game scene will some Treasures scattered on the ground, every treasure has a score, players operate game characters to pick up the Treasures of the earth, and then get the corresponding scores. 
 
# # installation and operation 
Install ` pomelo ` 
 
` ` ` bash 
NPM install - g pomelo 
` ` ` 
Access to the source code 
` ` ` bash 
Git clone https://github.com/NetEase/treasures.git 
` ` ` 
Install ` NPM ` rely on package (first into the project directory) 
` ` ` bash 
Sh NPM - install. Sh 
` ` ` 
Start ` web server - ` (enter first ` web server ` directory) 
` ` ` bash 
The node app. Js 
` ` ` 
Start ` game - server ` (enter first ` game - server ` directory) 
` ` ` bash 
Pomelo start 
` ` ` 
In the browser to access [http://localhost:3001] (http://localhost:3001) into the game. 
 
If any problem, can consult [pomelo quick start guide] (https://github.com/NetEase/pomelo/wiki/pomelo%E5%BF%AB%E9%80%9F%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97) 
 
Also can consult/tutorial1 distributed chat (https://github.com/NetEase/pomelo/wiki/tutorial1 - % E5 B8 E5 % % % 88% 88% 83% E5 E8 BC 8 f % % % % 81% A4 E5 8 a % % % A9) 
 
# # architecture 
Treasures is divided into two parts of the web Server and the game - Server. 
 
* ` web server - ` was established with Express the most a basic HTTP server, used to provide service for the browser page access. 
 
* ` game - server ` is WebSocket server, used to run the game logic. 
 
First of all, through a configuration file to see ` game, the concrete structure for server ` ` game - server/config/server json ` 
` ` ` javascript 
{ 
"Development" : { 
"Connector" : 
{" id ":" connector - server - 1 ", "the host" : "127.0.0.1", "port" : 3150, "clientPort:" 3010, "frontend" : true}. 
{" id ":" connector - server - 2 ", "the host" : "127.0.0.1", "port" : 3151, "clientPort:" 3011, "frontend" : true} 
]. 
"Area" : 
{" id ":" area - server - 1 ", "the host" : "127.0.0.1", "port" : 3250, "areaId" : 1} 
]. 
"Gate" : 
{" id ":" gate - server - 1 ", "the host" : "127.0.0.1", "clientPort:" 3014, "frontend" : true} 
] 
} 
} 
` ` ` 
You can see that the server is made from the following several parts: 
 
* 1 a ` gate ` server, mainly used for load balancing, to spread out from the client connected to two ` connector ` server. 
* 2 ` connector ` server, mainly used for receiving and sending messages. 
* a ` area ` server, mainly used for driving game scenarios and logic 
 
The relationship between the server, as shown in the figure below: 
 
! [treasure - arch] (http://pomelo.netease.com/resource/documentImage/treasure-arch.png) 
 
# # source code analysis 
# # # # # by the game process analysis code: 
 
# # # 1. Connect to the server 
Client ` web server/public/js/main of js ` ` entry ` method: 
 
` ` ` javascript 
Pomelo. Request (' gate. GateHandler. QueryEntry '{uid: name}, function (data) { 
/ /... 
}); 
` ` ` 
 
Server-side ` game server/app/servers/gate/handler/gateHandler of js ` ` queryEntry ` method: 
 
` ` ` javascript 
Handler. Prototype. QueryEntry = function (MSG, session, next) { 
/ /... 
/ / by CRC algorithm will players enter the user name string into an integer, then the connector number more than take a hash server to a connector on the server. 
/ / in the end will connect the connector of the server host and port returned to the client. 
Next (null, {code: code. OK, host: res., host, port: res. WsPort}); 
}; 
` ` ` 
 
So the client can connect to the assigned ` connector ` on the server. 
 
# # # 2. Enter the game 
After establishing a connection with ` connector ` server, to get into the game: 
 
` ` ` javascript 
Pomelo. Request (' connector. EntryHandler. Entry, {name: the name}, function (data) { 
/ /... 
}); 
` ` ` 
 
The first time the client to ` connector ` server request, server will ` session initialization and binding ` information: 
 
` ` ` javascript 
The session. The bind (playerId); / / the session with playerId binding 
Session. Set (" areaId ", 1); / / set the player areaId 
` ` ` 
And will the ` connector ` server's serial number and a variable on the ` id ` combined to generate the role of ` playerId `, finally will generate the ` playerId ` sent to the client. 
 
The client requests to the server initiated into the scene: a 
 
` ` ` javascript 
Pomelo. Request (area. PlayerHandler. "enterScene", {name: name, playerId: data. The playerId}, function (data) { 
/ /... 
}); 
` ` ` 
 
The client sends a request to the server requests arrive first ` connector ` server, then ` connector ` server according to the default of pomelo framework forwarding rules (` pomelo/lib/components/proxy of js ` ` defaultRoute ` function), and routes the request to the appropriate ` area ` server (in this case only a ` area ` server), ` area ` the server ` playerHandler ` reprocessing the corresponding request. This is added to the game scene. 
 
After a player to join the game scene, the other players must be able to see the players to join immediately, so the server must the message broadcast to all players in the game scene. 
Establish a ` channel `, all players will join in to join the game scene this ` channel ` 
 
` ` ` javascript 
/ / game server/app/models/area of js addEntity function 
/ / get the channel, if not create a, then the player to join the channel 
GetChannel (). The add (e.i d, e.s erverId); 
` ` ` 
 
When ` area of ` players to join, or other events is changed, the information will be pushed to in this ` channel ` each of the players. For example there are players to join: 
 
` ` ` javascript 
/ / game server/app/models/area of js entityUpdate function 
GetChannel (). PushMessage ({route: 'addEntities, entities: added}); 
/ / note: entityUpdate by game - server/app/models/timer. The tick in js function calls (see below). 
` ` ` 
 
These messages by ` connector ` server to the client. ` area ` messages in is through ` session. FrontendId ` to decide by which ` connector ` server sent out. 
 
The client accepts the message: 
` ` ` javascript 
/ / web server/public/js/msgHandler init function in js: 
/ / when there are new players to join, the server will broadcast message to all players. The client through this routing binding, to get messages. 
Pomelo. On (' addEntities', function (data) { 
/ /... 
}); 
` ` ` 
 
# # # 3. The Area to the server 
` area ` server is a ` tick ` to drive the game scene. Each tick (100 ms) of scenario ` entity ` status updates. If the ` entities ` state changed, the new state will be pushed to all related to the client. 
` ` ` javascript 
The function tick () { 
/ / the run all the action 
Area. ActionManager (). The update (); 
/ / update entities 
Area. EntityUpdate (); 
/ / update rank 
Area. RankUpdate (); 
} 
` ` ` 
 
For example the player initiated a ` move ` action: 
 
The client 
` ` ` javascript 
/ / send the service side move request request: 
Pomelo. Request (' area. PlayerHandler. Move, {targetPos: targetPos}, function (result) {...}); 
` ` ` 
The service side ` playerHandler ` accept requests: 
` ` ` javascript 
Handler. Move = function (MSG, session, next) { 
/ /... 
/ / create a move action 
Var action = new Move ({ 
The entity: player, 
EndPos: endPos 
}); 
/ /... 
/ / and the action to join her in actionManager: 
If (area. The timer (). AddAction (action)) {... }); 
/ /... 
}); 
` ` ` 
Then the ` action ` will next ` tick ` updates. 
 
# # # 4. The client to send and receive messages 
The client and server communication has the following several ways: 
 
* Request - Response 
 
` ` ` javascript 
/ / sends a request to the connector parameters {name: the name} 
Pomelo. Request (' connector. EntryHandler. Entry, {name: the name}, function (data) { 
/ / callback function to get the request to return the result 
/ / do something 
}); 
` ` ` 
 
* Notify (to send a notification of a service) 
 
` ` ` javascript 
/ / send the server-side notify notice 
Pomelo. Notify (" * * *. * * *. * * * ", params); 
` ` ` 
 
* Push (server-side initiative to send a message to the client) 
 
` ` ` javascript 
/ / when there are new players to join, the server will broadcast message to all players. The client through this routing binding, to get the message: 
Pomelo. On (' addEntities', function (data) { 
/ /... 
}); 
` ` ` 
 
# # # 5. Leave the game 
 
When players leave the game, ` connector ` server would receive disconnect message first. Then need to ` area ` server, user, and broadcast messages to other players online. 
 
Because the server process are independent, so it would involve a ` RPC ` calls. Fortunately, Pomelo framework for ` RPC ` did a very good packaging, such as: 
` area ` server to provide a series of ` remote ` interface for other server process calls, need only in ` servers/area ` directory to create a ` remote ` directory, in ` remote ` files in the directory exposed interface, can be as a ` RPC ` call interface. 
 
For example, the player leave: 
 
` ` ` javascript 
/ / in the connector to the session binding events, when the session is closed, the triggering event 
Session. On (' closed 'onUserLeave. Bind (null, the self. The app)); 
Var onUserLeave = function (app, session, a tiny) { 
If (session && session. The uid) { 
/ / RPC calls 
App. RPC. Area. PlayerRemote. PlayerLeave (session, {playerId: session. Get (" playerId "), areaId: session. Get (" areaId ")}, null); 
} 
}; 
` ` ` 
 
The corresponding ` area/remote/playerRemote ` in js ` playerLeave ` method 
 
` ` ` javascript 
Exp. PlayerLeave = function (args, cb) { 
/ /... 
/ / notice 
Area. GetChannel (.) pushMessage ({route: 'onUserLeave, code: consts. MESSAGE. The RES, playerId: playerId}); 
/ /... 
}; 
` ` ` 
Thus easily completed a call across processes. 


$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Google:

# Treasures Introduction
# # Description
[Treasures] (https://github.com/NetEase/treasures) game from [LordOfPomelo] (https://github.com/NetEase/lordofpomelo) drawn out, removed a lot of the game logic to more good show [Pomelo] (https://github.com/NetEase/pomelo) framework usage and operation mechanism .

Treasures is very simple, enter a user name , it will randomly get a game character and enter the game scene in the game scene will be scattered some of the treasures of the earth , every treasure has scores , players manipulate game characters to pick up the treasure, then to give the corresponding scores.

# # Install and run
Install `pomelo`

`` `bash
npm install-g pomelo
`` `
Get the source
`` `bash
git clone https://github.com/NetEase/treasures.git
`` `
Install `npm` dependencies ( first into the project directory )
`` `bash
sh npm-install.sh
`` `
Start `web-server` ( first into the `web-server` directory )
`` `bash
node app.js
`` `
Start `game-server` ( first enter the `game-server` directory )
`` `bash
pomelo start
`` `
In the browser to access [http://localhost:3001] (http://localhost:3001) into the game.

If you have questions , you can refer to [pomelo Quick Start Guide ] (https://github.com/NetEase/pomelo/wiki/pomelo% E5% BF% AB% E9% 80% 9F% E4% BD% BF% E7% 94 % A8% E6% 8C% 87% E5% 8D% 97)

You can also refer to [tutorial1 distributed chat ] (https://github.com/NetEase/pomelo/wiki/tutorial1--% E5% 88% 86% E5% B8% 83% E5% BC% 8F% E8% 81% 8A% E5% A4% A9)

# # Architecture
Treasures into web-Server and game-Server in two parts.

* `Web-server` is to use Express to establish a basis for most of the http server , the browser used to access the page to provide services .

* `Game-server` WebSocket server is used to run the entire game logic .

First, look through the configuration file `game-server` practical framework `game-server/config/server.json`
`` `javascript
{
  "development": {
    "connector": [
      {"id": "connector-server-1", "host": "127.0.0.1", "port": 3150, "clientPort": 3010, "frontend": true},
      {"id": "connector-server-2", "host": "127.0.0.1", "port": 3151, "clientPort": 3011, "frontend": true}
    ] ,
    "area": ​​[
      {"id": "area-server-1", "host": "127.0.0.1", "port": 3250, "areaId": 1}
    ] ,
    "gate": [
      {"id": "gate-server-1", "host": "127.0.0.1", "clientPort": 3014, "frontend": true}
    ]
  }
}
`` `
You can see , the server is composed of the following components:

* 1 `gate` server , mainly for load balancing , the connection from the client to both `connector` dispersed servers.
* 2 `connector` server to receive and send messages.
* An `area` server , mainly for driving game scenes and game logic

The relationship between the server , as shown below:

! [treasure-arch] (http://pomelo.netease.com/resource/documentImage/treasure-arch.png)

# # Source code analysis
# # # # # Through the game process to analyze the code :

# # # 1 . Connect to the server
Client `web-server/public/js/main.js` the `entry` method :

`` `javascript
pomelo.request ('gate.gateHandler.queryEntry', {uid: name}, function (data) {
  / / ...
} ) ;
`` `

Server `game-server/app/servers/gate/handler/gateHandler.js` the `queryEntry` method :

`` `javascript
Handler.prototype.queryEntry = function (msg, session, next) {
  / / ...
  / / Crc algorithm player input via the user name string is converted to an integer , then the number on the connector server modulo hash to a connector server.
  / / Finally the connector will connect the server host and port back to the client .
  next (null, {code: Code.OK, host: res.host, port: res.wsPort});
} ;
`` `

So that the client can connect to the assigned `connector` the server.

# # # 2 . Enter the game
With `connector` server connection is established , you can enter the game :

`` `javascript
pomelo.request ('connector.entryHandler.entry', {name: name}, function (data) {
  / / ...
} ) ;
`` `

The first time in the client `connector` server request , the server sends information `session` initialize and bind :

`` `javascript
session.bind (playerId); / / session binding with playerId
session.set ('areaId', 1); / / Set Player areaId
`` `
And the `connector` server serial number and a variable increment combinations to generate the `id` role `playerId`, finally generated this `playerId` sent to the client .

The client to the server initiates a request to enter the scene :

`` `javascript
pomelo.request ("area.playerHandler.enterScene", {name: name, playerId: data.playerId}, function (data) {
  / / ...
} ) ;
`` `

Client sends a request to the service , the request first reaches `connector` server, and then `connector` server 's default under the pomelo frame forwarding rules (`pomelo / lib / components / proxy.js` in `defaultRoute` function ) , will routes the request to the appropriate `area` server ( in this case there is only one server `area` ), `area` server `playerHandler` reprocessing corresponding request , so that players can join the game scene .

Players added to the game in a scene , the other players must be able to readily see the players to join , so the server must be the message broadcast to the scene in this game all the players .
Establish `channel`, all join this game players will be added to the scene this `channel` in

`` `javascript
/ / Game-server/app/models/area.js of addEntity function
/ / Get the channel, if not create one, and then the players to join the channel
getChannel (). add (e.id, e.serverId);
`` `

When the `area` of a player to join, or other event is changed , this information will be pushed in this `channel` for each player , such as when a player added :

`` `javascript
/ / Game-server/app/models/area.js of entityUpdate function
getChannel (). pushMessage ({route: 'addEntities', entities: added});
/ / Note : entityUpdate by game-server/app/models/timer.js the tick function calls ( see the next section ) .
`` `

These messages are sent through the `connector` server to the client . `Area` The message is `session.frontendId` to decide what `connector` is sent to the server .

Clients receive messages :
`` `javascript
/ / Web-server/public/js/msgHandler.js the init function :
/ / When a new player joins , the server broadcasts a message to all players . Clients through this route is bound to get the message .
pomelo.on ('addEntities', function (data) {
  / / ...
} ) ;
`` `

# # # 3. Area Server
`area` server is a `tick` driven game scenes , each tick (100ms) will be on the scene `entity` status to be updated. `entities` if the state has changed, then the new state will be pushed to all relevant clients.
`` `javascript
function tick () {
  / / run all the action
  area.actionManager (). update ();
  / / Update entities
  area.entityUpdate ();
  / / Update rank
  area.rankUpdate ();
}
`` `

For example, the player initiating a `move` action:

Client
`` `javascript
/ / Move to the service sends a request request:
pomelo.request ('area.playerHandler.move', {targetPos: targetPos}, function (result) {...});
`` `
`PlayerHandler` server accepts requests :
`` `javascript
handler.move = function (msg, session, next) {
  / / ...
  / / Generate a move action
  var action = new Move ({
    entity: player,
    endPos: endPos
  } ) ;
  / / ...
  / / And in the action added to actionManager :
  if (area.timer (). addAction (action)) {...});
  / / ...
} ) ;
`` `
Then the `action` a `tick` the next update .

# # # 4 . Clients to send and receive messages
Client and server communicate in the following ways :

* Request - Response mode

`` `javascript
/ / Send the request to the connector , the parameters {name: name}
pomelo.request ('connector.entryHandler.entry', {name: name}, function (data) {
  / / Callback function returns the result to obtain the requested
  / / Do something
} ) ;
`` `

* Notify ( send notifications to the service side )

`` `javascript
/ / To the service sends notify notify
pomelo.notify ("***. ***. ***", params);
`` `

* Push ( active server to send messages to the client )

`` `javascript
/ / When a new player joins , the server broadcasts a message to all players . Clients through this route is bound to get the message :
pomelo.on ('addEntities', function (data) {
  / / ...
} ) ;
`` `

# # # 5 . Leave the game

When a player leaves the game , `connector` server will first receive a disconnect message . Then need to `area` server will remove user and broadcast messages to other players online .
 
Because between the server processes are independent, so this involves a `RPC` calls. Fortunately Pomelo framework `RPC` done a very good package , for example:
`area` server you want to provide a series of `remote` interfaces for other server processes invoked only in the `servers / area` directory, create a `remote` directory, `remote` directory files exposed interfaces, can be used as `RPC` call interface .

For example, the player leaves :

`` `javascript
/ / Connector binding events in the session , when the session is closed, the trigger event
session.on ('closed', onUserLeave.bind (null, self.app));
var onUserLeave = function (app, session, reason) {
  if (session && session.uid) {
    / / Rpc call
    app.rpc.area.playerRemote.playerLeave (session, {playerId: session.get ('playerId'), areaId: session.get ('areaId')}, null);
  }
} ;
`` `

Corresponding `area / remote / playerRemote.js` method of `playerLeave`

`` `javascript
exp.playerLeave = function (args, cb) {
  / / ...
  / / Notice
  area.getChannel (). pushMessage ({route: 'onUserLeave', code: consts.MESSAGE.RES, playerId: playerId});
  / / ...
} ;
`` `
This easily completed a cross-process calls.

