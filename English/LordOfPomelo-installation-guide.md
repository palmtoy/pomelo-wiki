# LordOfPomelo introduction
 
LordOfPomelo is a MMORPG(massively multiplayer online role playing game) game demo which is developed based on Pomelo framework. It has many game features such as: character, monster, item, battle, chat, skill, upgrade system, task system, team, game copy, etc.
 
LordOfPomelo's server-side is developed based on Pomelo framework, the client-side uses colorbox framework which is based on HTML5. It has been finished in about just 3 months. The server-side is about 8000 lines of code, the client-side is about 6000 lines of code. More than 800 players can access a single scene concurrently, the response time is about 100ms. 
 
# runtime environment 
* [nodejs](http://nodejs.org/) 
* Windows, Linux, or MacOS
* MySql database 
 
# LordOfPomelo installation 
 
## Download the source code 
 
`git clone https://github.com/NetEase/lordofpomelo.git`
 
## Dependencies installation
 
Enter the directory: 
`cd lordofpomelo`
 
Install dependencies :
`sh npm-install.sh`(Windows: `npm-install.bat`)
 
# Create MySql database
 
## Create the database 
sql file path: ./game-server/config/schema/Pomelo.sql

* Install MySql database(abbreviated) 
* Login MySql: 
`mysql -u username -p password`
(successful login prompt: mysql>)
* Create a database: 
`mysql> create database Pomelo;`
* Select the database: 
`mysql> use Pomelo;`
* Import sql file: 
`mysql> source ./game-server/config/schema/Pomelo.sql;`
 
## Modify the database configuration
The database configuration file is "./shared/config/mysql.json"
```json
{
  "development": {
   "host" : "127.0.0.1",
    "port" : "3306",
    "database" : "Pomelo",
    "user" : "xy",
    "password" : "dev"
  },
  "production": {
   "host" : "127.0.0.1",
    "port" : "3306",
    "database" : "Pomelo",
    "user" : "xy",
    "password" : "dev"
  }
}
```
 
Change the "development" environment database configuration to the actual configuration.
 
# Run the game
Start the game-server and web-server separately.
game-server's startup method:
 
* `pomelo start` (pomelo installation [pomelo quick start guide](https://github.com/NetEase/pomelo/wiki/pomelo快速使用指南)) Note: If pomelo processes don't exit completely, you can use `pomelo kill - force` to end all node processes.
 
web-server's startup method:
 
* `cd web-server && node app`
 
# Visit the game
You can run servers locally, visit http://localhost:3001 or http://127.0.0.1:3001 directly in the browser.
 
Browser must support websocket. We recommend chrome.
 
# Related problem solution
1. Port conflicts 
 
Modify the server configuration file "./game-server/config/servers.json", reads as follows: 
```json
{
  "development": {
    "connector": [
      {"id": "connector-server-1", "host": "127.0.0.1", "port": 3150, "clientPort": 3010, "frontend": true},
      {"id": "connector-server-2", "host": "127.0.0.1", "port": 3151, "clientPort":3011, "frontend": true}
    ],
    "area": [
      {"id": "area-server-1", "host": "127.0.0.1", "port": 3250, "area": 1}, 
      {"id": "area-server-2", "host": "127.0.0.1", "port": 3251, "area": 2}, 
      {"id": "area-server-3", "host": "127.0.0.1", "port": 3252, "area": 3}, 
      {"id": "instance-server-1", "host": "127.0.0.1", "port": 3260, "instance": true},
      {"id": "instance-server-2", "host": "127.0.0.1", "port": 3261, "instance": true},
      {"id": "instance-server-3", "host": "127.0.0.1", "port": 3262, "instance": true}
    ],
    "chat": [
      {"id":"chat-server-1","host":"127.0.0.1","port":3450}
    ],
    "path": [
      {"id": "path-server-1", "host": "127.0.0.1", "port": 3550}
    ],
    "auth": [
      {"id": "auth-server-1", "host": "127.0.0.1", "port": 3650}
    ],
    "gate": [
      {"id": "gate-server-1", "host": "127.0.0.1", "clientPort": 3014, "frontend": true}
    ],
    "manager": [
      {"id":"manager-server-1","host":"127.0.0.1","port":3750}
    ]
  },
  "production": {
    // ...
  }
}
```
 
The configuration file defines various servers' configuration information in development and production environments, including the server type, address and port number, etc. The parameters of the production environment has the similar structure as development environment. When the "frontend" parameter is "true", means this is a front-end server. You can modify the corresponding port number when it conflicts with others. 
 
You can modify the file "./web-server/public/js/config/config.js" either, reads as follows: 
```json
__resources__["/config.js"] = {meta: {mimetype: "application/javascript"}, data: function(exports, require, module, __filename, __dirname) {
  module.exports = { 
    IMAGE_URL:'http://pomelo.netease.com/art/',
    GATE_HOST: window.location.hostname,
    GATE_PORT:3014
  };
}};
```
This file is the common configuration information used in the web server: IMAGE_URL is the address of the images and other static resources; GATE_HOST is the entrance of a gate-server to accept the websocket connections; Firstly, all the websocket requests must get the connector-server id via the gate-server, and then establish a connection with the connector-server; GATE_PORT is the the port number of the front-end gate-server. Manager-server is mainly responsible for copy and team functions. 
 
# Configuration file summary description 
* ./game-server/config/master.json
 
This is the master server configuration information, including the server address and port number under development and production environments. 
 
The master server is responsible for starting, stopping the servers, and monitoring all servers' status information. 

* ./game-server/config/servers.json
 
It contains area-server, connector-server's configuration information, including server address and port number under the development and production environments. As the connector-server is the front-end server, used to receive the request and forward players' request, so there will be a clientPort. 
 
* ./shared/config/mysql.json
 
This is database configuration information. After the installation of LordOfPomelo, you need to modify the parameters of the development and production environments according to the actual situation of the database installation. 
 
* ./web-server/public/js/config/config.js
 
This is a configuration file with the client images and other static resources and HTTP access address. 
 
# Login problem description
LordOfPomelo provides a new user registration and using github, google, facebook, twitter and weibo in the authorized manner to login. When using github or other authorized manner to login, you need to authenticate themselves through OAuth authorization, then modify corresponding information in the configuration file(./web-server/config/oauth.json).

