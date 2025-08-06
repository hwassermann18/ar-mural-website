A standard feature of any content creation tool is the ability to create, delete, and manage multiple projects. Since the original AR Mural contained only one universal drawing space, the current implementation is the beginning of this exploration. 

## **Data Storage & Server**

Originally, for the one drawing space, we had one LevelDB instance with 

- Key: (x,y) coordinate of a "chunk" in the AR space, containing drawing objects (e.g. brushstrokes, shapes, text)
- Value: JSON file of all objects and their metadata (size, color, type, etc.)

The idea of multiple LevelDB instances, where each database would correspond to one mural, was explored, but abandoned upon the realization that more than one LevelDB cannot be open at once. 

Instead, the original key was adjusted to have a mural-id prefix, where

- New Key: (mural_id:x,y)

And chunk retrieval would be based on this prefix. 
However, each mural does have its own "group" of MQTT Server topics prefixed with the mural ID. For example, when creating an object in mural 1, `ClientManager` would send a message to `mural_1/cmd/client_id`. 


## **Client-side/UI**

- Once a specific "mural button" is clicked in the Mural Menu, its chosen muralID is stored during transfer to the Main scene

- In the Main scene, that stored mural id is used to send a fetch request to the server, which sends back mural-specific data which is loaded into the scene.


