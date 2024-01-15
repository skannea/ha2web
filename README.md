This is an instruction on how to build **web applications based on entities** that are managed by your Home Assistant (HA) installation.

**This is not a complete development environment** for web applications. 

This is only a few simple source code files that are easy to understand and modify for your needs.

# Prerequisites
- An **MQTT broker** (for example, the Mosquitto add-on) must be installed and set up for communication between HA and clients (browsers) running the application.
- The **HA web server** folder */config/www* must exist. Application web pages are placed there.
- The **python_script** functionality must be enabled in *configuration.yaml* (by adding `python_script:` ).
- **Security** aspects must be considered if your HA can be accessed from internet. See later in this document.

# How it works
- The application web page contains code for an MQTT client.
- A HA automation triggers when the state changes for one of a several entities in an entity list and publishes an MQTT message about this. 
- When a browser loads the application web page, it starts subscribing for such messages. 
- When a message is received, it is possible to act on it, for example changing color on an icon or displaying a value.
- It is possible to publish MQTT messages on user input, such as when the user of the web application clicks a button.
- Each such message holds data on *entity id* and what HA service to call, for example *switch.turn_on* or *input_number.set_value*. 
- The HA automation subscribes for such messages. When a message is received, the automation makes the service call.

On an application level, the **HA automation acts as a server that may serve multiple clients**. Each client has loaded the application web page and started the subscription for messages.

**Messages to the client** (from the HA automation) holds not only state of the changed entity, but the current state of all entities in the list handled by the HA automation. 
The messages are published with the *retain flag*. This makes the MQTT broker store the message to be sent to new subscribers as a first message. 
So, when the application starts and subscribes for messages, all states are available in the very first message.

**Messages from the client** (to the HA automation) contain an *entity_id* that is checked against the list of entities. Entities that are not in the list are ignored.

# Messages
## From HA to client

The MQTT payload is an array coded as a json string.

`array[0][0]` is *entity_id* of the entity that caused the message

`array[0][1]` is the state of that entity

`array[1][0]` is *entity_id* of the first entity in the list

`array[1][1]` is the state of that entity

`array[2][0]` is *entity_id* of the second entity in the list 

`array[2][1]` is the state of that entity

and so on.

## From client to HA

The MQTT payload is a dictionaty coded as a json string:

`{ "domain":DOMAIN, "service":SERVICE, "input":{'entity_id':ENTITY, OTHER } }`

where 
- DOMAIN is the domain of the entity, for example `"light"`
- SERVICE is the service to run, for example `"turn_on"`
- ENTITY is the *entity_id* of the entity, for example `"light.garden"`
- OTHER are other, required or optional data for the service

An example   

`{ "domain":"light", "service":"turn_on", "input":{ "entity_id":"light.garden", "brightness":"125", "transition":"1500" } }`



# The files
## ha2web_blueprint.yaml
`/config/blueprints/automation/ha2web/ha2web_blueprint.yaml`

This is an HA automation blueprint file. Create an automation based on this file:
- specify the list of entities to be used by the application
- specify the MQTT topics to be used for communication
The automation will send entity state changes to the client and forward service calls from the client.

## ha2web_script.py
`/config/python_scripts/ha2web_script.py`

A one line python_script file that makes it possible to forward calls from the client to HA.

`hass.services.call( data['domain'], data['service'], data['input'], False )`

## ha2web_example.html
`/config/www/ha2web_example.html`

An example HTML file that can be used as a template or as a start point for your own files. It also contains the required JavaScript code. 
The example is made in a way that works for a simple application. For a more complex applications 

# Coding

## Messages from HA

- When the client starts it makes a connection to the MQTT broker.
- When connection is established, the client subscribes for messages.
- When the first message arrives it is handled by `onFirstMessage()` that calls `hassInput()` for each entity in the message.
- All following messages are handled by `onMessage()` that calls `hassInput()` for the changed entity in the message.
- From the input to `hassInput()`, string variables `entity` and `state` are updated.
- Now relevant parts of the HTML page can be updated.

        function hassInput( data ) { 
          var entity = data[0];
          var state  = data[1];
          
          switch(entity)  {
            case 'sensor.outdoor_temp':
            document.getElementById('outtemp').innerHTML = "It is " + state + " centigrades outside.";
            break;    

            case 'light.garden':
            document.getElementById('gardenlamp').style.color = (state == 'on') ? 'red' : 'black';
            break;    
          }   
        }  



