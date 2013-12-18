# Introduce various types of server
 
LordOfPomelo adopts the distributed design. The service side is composed of a cluster of servers, including: many scene servers, one or more pathfinding servers, chat server, copy/team server, global server, long connection server, etc. 
 
![backend server](http://pomelo.netease.com/resource/documentImage/lordofpomelo/servers.png)
 
### Connector server 
 
Different from web server with short connections, the connections between the clients and servers are long connections in online games. The long connection itself is need certain resources to maintain. In LordOfPomelo, we use the websocket protocol to establish connection between the client and the server. The connector server is used to maintain the connections, and transfer messages between the clients and the servers. In LordOfPomelo, the connection state between client and server is maintained by an abstract session. The session is a client's passport on the service end, used to maintain the user's login status, the user's basic information, and the user's connection information. 
 
### Gate server 
 
Gate server's main function is to provide a unified websocket entrance to users, and is responsible for user authentication and the distribution of the connector-servers. Unlike the connector-servers, there is only one gate-server in LordOfPomelo. Gate server will expose a fixed websocket interface to all the clients. When a user logs in, the user will connect to the gate-server firstly.After completing the verification, the user will obtain corresponding connetor-server's information assigned by the gate-server. Later, the client will disconnect from the gate-server. The client will connect to the corresponding connector-server and obtain the corresponding services. 
 
### Authentication server 
 
This server is responsible for the user registration and authentication. It is the unified entrance of user authentication, provides remote call interface for user authentication to other servers. This server's main purpose is to shield the details of certification authentication, provide unified authentication interface to other servers. 
 
### Scene server
 
In online games, for performance and load considerations, the big game world will always be divided into multiple regions. These different regions are scenes. In LordOfPomelo, a game map is corresponding to a game scene, and corresponding to a separate scene-server. The scene is the basic unit to constitute LordOfPomelo game world, it cannot be divided and extended parallelly. Scene-server is responsible for maintaining all entities in the scene, and driving entities' AI and running game logic. 
 
Scene-server is responsible for handling almost all the game logic, at the same time, providing interfaces of scene data manipulation to other servers. In LordOfPomelo, although the scene itself can not be separated, but you can add new scene to distribute users, so as to improve the total load of the game-server. And some of the scene related services can be extended by running independent services. 
 
### Pathfinding server 
 
Pathfinding service is one of the basic game server service. Player movement, monster movement need pathfinding services to support. Its function is calculating an optimal path according to the starting point and end point in the map. Due to the pathfinding is a typical stateless, compute-intensive service, in LordOfPomelo, we separate the pathfinding logic from the scene logic. We start a pathfinding server, so as to reduce the pressure of the scene-server. The pathfinding server can be extended parallelly according to necessity. LordOfPomelo uses A* pathfinding algorithm implementation, provides a common computing interface. The pathfinding algorithm is encapsulated as a module. Specific information, see [pomelo-pathfinding](https://github.com/NetEase/pomelo-pathfinding). 
 
### Chat server 
 
Chat service is one of the basic services of online game. In LordOfPomelo, chat service is separated from scene services. We start an independent chat server. Chat server maintains a list of all online users. Chat server communicates with connector-server by the list, so as to achieve instant communication between players. 
 
### Copy/team server 
 
Copy/team server is one of the back-end server cluster members. It is responsible for managing the whole life cycle of copies and team related operations globally. The specific reference [lordofpomelo 0.3 new features](https://github.com/NetEase/pomelo/wiki/lordofpomelo-0.3%E6%96%B0%E7%89%B9%E6%80%A7)
 
## Scene management and introduction 
 
LordOfPomelo's scene service is responsible for managing all the entities and the game world. But for reasons of safety and consistency, all game logic validation is carried out on the back-end servers, so the scene service also includes all game logic judgment and processing. LordOfPomelo's scene service is the core of the game service, also the most complicated part of logic. The following is to analyze LordOfPomelo's scene service: 
 
### Scene management in the LordOfPomelo 
 
Each scene in LordOfPomelo corresponds to a separate scene server. All of the business logic is processed by the scene server. A global server is responsible for operatings across scenes. Below is the function of a single scene-server: 
 
![area](http://pomelo.netease.com/resource/documentImage/lordofpomelo/area.png)
 
#### Entity management 
 
All the entities in the game scene will be loaded into memory when they enter the scene. After that, all the changes of entities will take place in memory directly. The changes will be synchronized to the database by the data timing synchronization module. Under this design, all data operations in the scene will be taken in memory directly. It improves the overall performance of LordOfPomelo and avoids the frequent slow IO operations. 
 
In LordOfPomelo, the code of entity definition is in the "domain/entity" folder. The definition adopts the object-oriented design thought. All entities are inherited from the entity class. Adding or deleting entities operatings in the area are performed by the interfaces(addEntity/removeEntity). It lightens the burden of management of the objects. 
 
#### Message service 
 
In LordOfPomelo, message service can be divided into two types: one-on-one RPC(request<-->response), as well as the one-to-many broadcast message. 
In LordOfPomelo, since logic verification is performed on the server side. For most of the client's requests, the server will send a direct response. This is just similar to the "request/response" mode in the web. 
 
In LordOfPomelo, broadcasting is a service based on AOI area notification. After receiving broadcast messages, LordOfPomelo will get the players list who should to be informed depending on the type of message by the AOI service. And LordOfPomelo will broadcast messages to the players on the list. This reduces the number of the message significantly.
 
#### AOI service 
 
The player's field of vision is generally much smaller than the size of the scene. For most of the messages in the scene, broadcasting them simply to all of the entities in the scene is not necessary and can't withstand. When a message needs to be sent, how to determine the players list who should to be informed is the function of the AOI module . 
 
LordOfPomelo's AOI service is implemented based on the tower model. The basic method is to divide the whole map into a number of same size towers. Each tower is responsible for maintaining a list of objects which contains all objects within the scope of the tower. Then we trigger various AOI events on the basis of the data structure.

