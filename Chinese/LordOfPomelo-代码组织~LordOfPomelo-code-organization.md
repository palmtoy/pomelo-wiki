#LordOfPomelo代码组织

LordOfPomelo的代码主要包括两部分: 后端服务器代码game-server和前端客户端代码web-server. game-server是游戏服务端, 包括所有的游戏逻辑代码和游戏服务器代码. web-server是游戏客户端, 包括用户注册和登录界面代码, 以及一个用HTML5编写的游戏客户端. 除了这两部分之外, 还有一个公用的shared目录, 用来存放前后端共用的代码和配置. 

作为一个分布式的游戏服务器, LordOfPomelo可以同时运行在多台服务器上, 却统一使用同一套代码, 不同的服务器会根据配置加载各自的目录代码. 下图是LordOfPomelo的代码结构: 

![lordofpomelo_arch](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/LOP_arch.png)

## game-server服务端代码分析

![game-server](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/game-server.png)

game-server根目录下的app.js是服务器代码的入口, 其他目录的功能如下: 
* /app    : 服务端js代码, 包括服务器代码和游戏逻辑代码. 
* /config : LordOfPomelo中的配置文件. 
* /logs   : 服务器端运行时产生的日志文件. 
* /scripts: 统计模块对应的本地脚本. 
* /test   : LordOfPomelo中的测试用例

### 逻辑代码

逻辑代码主要用来完成具体的业务逻辑, 如用来驱动怪物的AI代码, 用来计算地图中路径的寻路代码等, 逻辑代码在/app/domain目录下: 

![logic](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/logic.png)

* /action : 负责处理客户端的请求. 由于场景是由tick驱动的, 而tick的间隔一般较短(默认100ms), 当请求需要在多个tick中执行的时候就会被封装为一个action来执行. 
* /aoi    : aoi相关逻辑, 包括aoi消息的封装, 以及对aoi消息的处理. 
* /area   : 场景相关逻辑, 提供场景中的主要接口. 包括: 场景中实体的加入、更新和删除, 广播消息的推送, 场景中服务的访问(AOI, AI等), 场景信息的获取等. 同时还包括一个Timer, 用来驱动场景中的逻辑. 
* /entity : 场景中的所有实体, 包括玩家, 怪物, npc, 宝物, 装备, 队伍等. 
* /event  : 用来集中处理场景逻辑中产生的各种事件, 包括玩家消息, 怪物消息等. 
* /map    : 用来完成地图的加载和解析, 以及地图中区域的抽象. 
* /task   : 任务相关的代码, 控制任务的执行和取消, 以及任务奖励的获得. 

### 服务器代码. 

![servers](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/servers.png)

服务器代码在/servers目录下, 通过规约的形式组织, 对外提供rpc接口, 处理客户端和服务端的请求并返回结果. LordOfPomelo中使用的服务器包括: 
* /area     : 场景服务器, 用来储存场景信息, 处理客户端的请求, 如用户添加, 删除, 攻击等操作. 
* /chat     : 聊天服务器, 处理聊天信息
* /connector: 连接服务器, 负责维护用户session, 接受用户数据, 并将服务端的广播数据推送给玩家
* /login    : 登录服务器, 用来验证用户登录信息
* /path     : 寻路服务器, 用来完成路径计算功能. 
* /manager  : 副本/组队服务器, 用来管理全局的副本和组队功能. 

## web-server代码架构

![web server](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/web_server.png)

LordOfPomelo的页面端代码主要分为两个部分: 基于HTML5开发的ui代码和使用colorbox开发的游戏逻辑代码. ui代码包括注册/登录页面, 游戏场景中的各种选项和菜单. 这些代码基于HTML5开发, 使用css3进行渲染. 游戏场景的绘制和游戏逻辑的驱动则是基于colorbox开发, 并使用到了HTML5中的很多特性. 

除此之外, web-server中还包括用户注册代码和oauth验证的逻辑, 这些代码在lib目录下. 

下图是页面端的内容: 

![client & html](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/client_html.png)

* /animation_json : 动画相关的json描述.
* /css            : 代码中所用到的css文件. 
* /image          : 客户端中用到的图片资源.
* /js             : 所有客户端的js文件.
* /index.html     : 是LordOfPomelo的入口文件.

下图是游戏js代码组织: 

![client & html](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/game_client.png)

* /config   : 客户端的配置信息.
* /handler  : 客户端的handler, 用来处理服务端response请求.
* /lib      : colorbox和pomelo的客户端通讯库代码.
* /model    : 客户端的游戏逻辑代码.
* /ui       : ui代码.
* /utils    : 客户端用到的工具类.
* /app.js   : 客户端的初始化入口, 负责初始化客户端的逻辑代码. 


$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$


# LordOfPomelo code organization 
 
LordOfPomelo's code mainly includes two parts: the back-end server code(game-server) and the front-end client code(web-server). The game-server is a game server, including all of the game logic code and the server code. Web-server is the game client, including user registration and login interface code, as well as a game client written in HTML5. In addition to these two parts, there is a common shared directory, used to store shared code and configuration of front-end and back-end.
 
As a distributed game server, LordOfPomelo can run on multiple servers at the same time, but using the same set of code, different server will load respective directories according to the configuration. Below is the code structure of the LordOfPomelo: 
 
![lordofpomelo_arch](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/LOP_arch.png) 
 
## game-server server code analysis 
 
![game-server](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/game-server.png) 
 
The app.js in the root directory of game-server is the entrance of the server code, the functions of the other directories are as follows: 
* /app: server-side js code, including server code and game logic code. 
* /config: the configuration file of LordOfPomelo. 
* /logs: server-side log files which are generated at runtime.
* /scripts: the local script for the statistics module. 
* /test: test cases of LordOfPomelo.
 
### logic code 
 
Logic code is mainly used to complete the specific game logic, such as code used to drive the monster's AI, the pathfinding code used to calculate the map path. Logic code is in the "/app/domain" directory: 
 
![logic](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/logic.png) 
 
* /action: responsible for dealing with the client's requests. The scene is driven by tick, generally, the interval of tick is short(default 100ms). When the request need to be performed in more than one tick, it will be encapsulated as an action to perform. 
* /aoi: aoi relate logic codes, including aoi message encapsulation, and handling of aoi messages. 
* /area: scene related logic codes, provides the main interfaces in the scene. Including: add, update, and delete entities in the scene, broadcast messages, provide services in the scene (AOI, AI etc.), provide scene information, etc. Also includes a timer, used to drive the logic in the scene. 
* /entity: all entities in the scene, including the player, monster, NPC, treasure, equipment, team, etc. 
* /event: processing logic for a variety of events in the scene, including the player's and monster's messages, etc. 
* /map: used to complete the loading and parsing the maps, and abstract the area in the map. 
* /task: task related code, controling task execution and cancellation, obtaining the quest rewards. 
 
### server code
 
![servers](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/servers.png) 
 
Server codes are in "/servers" directory, organized through the statute, provide the RPC interface, deal with the client and server requests and return the results. Servers used in LordOfPomelo including: 
* /area: scene server, used to store the scene information, dealing with the client's request, such as to add/delete users, to attack monsters, and so on. 
* /chat: chat server, processing chat messages.
* /connector: connector server, responsible for maintaining the user session, accept user data, and push the server-side broadcasting data to the players. 
* /login: login server, used to verify the user login information
* /path: pathfinding server, used to complete the path calculation function. 
* /manager: copy/team server, which is used to manage global copy and team functions. 
 
## web-server code architecture 
 
![web server](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/web_server.png) 
 
LordOfPomelo's client-side code is divided into two main parts: UI code which is based on the HTML5 and game logic code which is based on colorbox. The UI code includes registration/login page, various options and menus in the game scene. These codes are developed based on HTML5, using CSS3 render the game scene. Game scene drawing and the logical driving are developed based on the colorbox, and use many features in HTML5. 
 
In addition, web-server also includes user registration code and oauth logic code which are in the lib directory. 
 
Below is the contents of the client:
 
![client & HTML](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/client_html.png) 
 
* /animation_json: animation related json description files. 
* /css: css files which are used in the code. 
* /image: the picture resources used in the client. 
* /js: all client js file. 
* /index.html: the entrance of the LordOfPomelo. 
 
Below is the game js code organization: 
 
![client & HTML](http://pomelo.netease.com/resource/documentImage/lordofpomelo/code/game_client.png) 
 
* /config: client configuration information. 
* /handler: client handler, used to handle the response of the server. 
* /lib: colorbox and pomelo client communication library code. 
* /model: the game logic code for the client. 
* /ui: ui code. 
* /utils: the tool classes used in the client. 
* /app.js: the entrance of the client initialization, is responsible for the initialization of the logic code for the client. 

