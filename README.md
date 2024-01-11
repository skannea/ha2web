This is an instruction on how to build **web applications based on entities** that are managed by your Home Assistant (HA) installation.

**This is not a complete development environment** for web applications. 

**This is the opposite:** a few simple source code files that are easy to understand and modify for your needs.

# Prerequisites
- An **MQTT broker** (for example, the Mosquitto add-on) must be installed and set up for communication between HA and clients (browsers) running the application.
- The **HA web server** must be enabled in *configuration.yaml*. Application web pages are placed in */config/www*.
- The **python_script** functionality must be enabled in *configuration.yaml*.
- **Security** aspects must be considered. See later in this document.

# How it works
- A HA automation triggers when the state changes for one of a several entities in an entity list and publishes an MQTT message about this. 
- The application web page contains code for an MQTT client.
- When a browser loads the application web page, it starts subscribing for such messages. 
- When a message is received, it is possible to act on it, for example changing color on an icon or displaying a value.
- It is possible to publish MQTT messages on user input, such as when the user of the web application clicks a button.
- Each such message holds data on *entity id* and what HA service to call, for example *switch.turn_on* or *input_number.set_value*. 
- The HA automation subscribes for such messages. When a message is received, the automation makes the service call.

On an application level, the **HA automation acts as a server that may serve multiple clients**. Each client has loaded the application web page and its own subscription for messages.

Messages to the client (from the HA automation) holds not only state of the changed entity, but the current state of all entities in the list handled by the HA automation. 
The messages are published with the *retain flag*. This makes the MQTT broker store the message to be sent to new subscribers as a first message. 
So, when the application starts and subscribes for messages, all states are available in the very first message.

Messages from the client (to the HA automation) contain an *entity_id* that is checked against the list of entities. That means that no other entities may be affected.

# The files
## ha2web_blueprint.yaml
This is an automation blueprint file. An automation is created by specification of the entities to be used by the application and the MQTT topics to be used for communication.
The automation will send entity state changes to the application and forward service calls from the application.
## ha2web_script.py
A python_script file that makes it possible to forward calls from the application to HA.
## ha2web_example.html
A HTML file that also contains the required JavaScript code. 




for the to control s as a serviceis 

|file       |  description      | box behavior  |
|-------------|-----------------|---------------|
||
automa|show state as colored icon|
