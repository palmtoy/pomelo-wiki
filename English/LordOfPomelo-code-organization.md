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

