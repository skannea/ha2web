Build a web application based on entities managed by your Home Assistant (HA) installation.
This project show how to do it.

**It is not** a high level environment for productive web application development. 
**It is the opposite:** a few simple source code files that are easy to understand and modify for your needs.

# Requirements
An MQTT broker must be installed for communication between HA and clients (browsers) running the application.
The HA web server must be enabled in order to return the application web page to the client.
The python_script functionality must be enabled. 

# How it works
An automation reacts on an entity state change and publishes an MQTT message. Clients, that are running the application, subscribes for the message. 
When received, it is up to the application to decide what to do. Typically it is reflected in the page, for example as an icon changing color.

It is also up to the client to react on various events and publish MQTT messages. Each message holds data, including *entity id* for a HA service call. 
Event are typically user input, like clicking a button or making a selection, but may also be timeouts or external events detected by the application.
The automation subscribes for such messages. When a message is received, the automation makes the service call using a python_script. 
Service calls are for example *switch.turn_on* or *input_number.set_value*. 

# Start up conditions
Messages from HA holds not only state of the changed entity. It holds the current state for all entities relevant for the application. 
The messages are also published with the *retain flag*. This makes the MQTT broker store the message to be sent to new subscribers as a first message. 
So, when the application starts and subscribes for messages, all states are available in the first message.

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
