# Home Assistant goes to Web - ha2web

This is an instruction on how to build **web applications based on entities that are managed by your Home Assistant** (HA) installation.

**This is not a complete development environment** for web applications. 

This is only a few simple source code files that are easy to understand and modify for your needs.

# Prerequisites
- An **MQTT broker** (for example, the Mosquitto add-on) must be installed and set up for communication between HA and clients (browsers) running the application.
- The **HA web server** folder */config/www* must exist. Application web pages are placed there.
- The **python_script** functionality must be enabled in *configuration.yaml* (by adding `python_script:` ).
- **Security** aspects must be considered if your HA installation can be accessed from the internet. See later in this document.

# How it works
## HA automation
- A HA automation (based on the HA2Web blueprint) is set up.
- It defines a *list of entities* that are involved in the web application.
- It triggers when the state changes for one of the entities in the *list of entities* and publishes an MQTT message about this.
- It triggers on MQTT messages that can be translated into entity service calls (such as *light.turn_on*).
- The entity service call is made if the entity matches the *list of entities*.
  
## Application web page
- The application web page contains code for an MQTT client.
- When the application web page is loaded, it starts subscribing for messages about state changes.
- When a message is received, it is possible to act on it, for example changing color on an icon or displaying a value.
- The application web page contains code that acts on user input, such as button clicks.
- The code makes MQTT client publish MQTT messages that can be translated into entity service calls.

# Messages
## From HA to client
Messages to the client (from the HA automation) holds not only state of the changed entity, but the current state of all entities in the list handled by the HA automation. 
The messages are published with the *retain flag*. This makes the MQTT broker store the message to be sent to new subscribers as a first message. 
So, when the application starts and subscribes for messages, all states are available in the very first message.

The MQTT payload is an array coded as a json string.

`array[0][0]` is *entity_id* of the entity that caused the message

`array[0][1]` is the state of that entity

`array[1][0]` is *entity_id* of the first entity in the list

`array[1][1]` is the state of that entity

`array[2][0]` is *entity_id* of the second entity in the list 

`array[2][1]` is the state of that entity

and so on.

## From client to HA
**Messages from the client** (to the HA automation) contain an *entity_id* that is checked against the list of entities. Entities that are not in the list are ignored.

The MQTT payload is a dictionaty coded as a json string:

`{ "domain":DOMAIN, "service":SERVICE, "input":{'entity_id':ENTITY, OTHER } }`

where 
- DOMAIN is the domain of the entity, for example `"light"`
- SERVICE is the service to run, for example `"turn_on"`
- ENTITY is the *entity_id* of the entity, for example `"light.garden"`
- OTHER are other, required or optional data for the service

An example   

`{ "domain":"light", "service":"turn_on", "input":{ "entity_id":"light.garden", "brightness":"125", "transition":"1500" } }`

## Multiple clients
On an application level, the HA automation acts as a server that may serve multiple clients. Each client has loaded the application web page and started the subscription for MQTT messages. A state change will affect all client. 



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

# Application web page 

## HTML code
The page HTML code in this example is kept as simple as possible. Only the body part is shown.

      <body onLoad="start()">
        <h1>Home assistant goes to web - ha2web example</h1>
        <div id="userinput">
          <button id="bedlamp">Bed lamp</button><br>
        </div>
        <div id="outtemp">text about temperature goes here</div> 
      </body>
      
- `<body onLoad="start()">` will load a function that initiate MQTT and handling of user input from elements in `<div id="userinput">`.
- `<button id="bedlamp">` is such an element. In this case the button will also change due to state change.
- `<div id="outtemp">` is a place to put a text about temperature.     

## Act on messages from HA
- When the client starts it makes a connection to the MQTT broker.
- When connection is established, the client subscribes for messages.
- When the first message arrives it is handled by `onFirstMessage()` that calls `hassInput()` for each entity in the message.
- All following messages are handled by `onMessage()` that calls `hassInput()` for the changed entity in the message.
- The input to `hassInput()` will update string variables `entity` and `state`.
- Now relevant parts of the HTML page can be updated.

        function hassInput( data ) { 
          var entity = data[0];
          var state  = data[1];
          
          switch(entity)  {
            case 'sensor.outdoor_temp':
            document.getElementById('outtemp').innerHTML = "It is " + state + " centigrades outside.";
            document.getElementById('outtemp').style.color = ( int(state) < 0 ) ? "blue" : "green";
            break;    

            case 'light.bedlamp':
            document.getElementById('bedlamp').style.background_color = (state == 'on') ? 'gold' : 'grey';
            break;    
          }   
        }  

## Act on input from user
- Domains that have entities that can be controlled by the user are for example: *script*, *light* and *input_number*.
- Such entities may be controlled using HTML elements that give an event. 
- The event is handled by `userInput()` that will send a message.
- When received by HA, the message will be carried out as a HA service call.

      function userInput(e) { 
          let id = e.target.id; // id of element that caused the event 
          let command = false;  // will be updated to send a message
          switch( id ) {
            case 'bedlamp':
            command = { "domain":"light", "service":"toggle",
                        "input":{'entity_id':'light.bedlamp'} } ;
            break;
          }
          if (command) mqttclient.send(  fromClientTopic, JSON.stringify( command ) );
     }      

## Let state changes update the page
Some input HTML elements, such as buttons, will normally not change the page when clicked.
Others, such as sliders and selections, do change the page.

If the entity state change is to be shown, the state change caused by the input must update the page. 
This is important, especially when there are more than one client handling the same entity.

See also the example under Attributes.
 
## Attribute sensors
A HA service call may affect also entity attributes.  
For example, when there is a slider for the brightness of a light, a message with the new level is sent to HA when the slider is moved. 

However, in messages from HA to the client, only the state is present, not the attribute values, such as brightness.
In order to change the page when the attribute value changes, a template sensor reflecting the attribute value is required. The template sensor is added to the list of entities in the automation.

Example with a template sensor with entity_id *sensor.garden_brightness*. 
The sensor uses the *brightness* attribute of *light.garden* to give a value 0 - 255.

Create it:

*Settings -> Devices & Services -> Helpers -> Create Helper -> Template -> Template Sensor -> Template a sensor*

Enter:
- Name : `Garden Brightness`
- State template :  `{{ (state_attr("light.garden","brightness") | float(0) ) | round(0)  }}`

The `float(0)`gives value 0 when the value is undefined or missing.

A code example where entities `light.garden` and `sensor.garden_brightness` are used. There page contains a button for toggling the light, a slider for the brightness and a button to set brightness to maximum. 

`userInput()`:
            
        case 'garden_toggle':
        command = { "domain":"light", "service":"toggle","input":{'entity_id':'light.garden'} } ;
        break;
        case 'garden_slider':
        let lev = document.getElementById(id).value;  
        command = { "domain":"light", "service":"turn_on","input":{'entity_id':'light.garden', 'brightness': lev } } ;
        break;
        case 'garden_full':
        command = { "domain":"light", "service":"turn_on","input":{'entity_id':'light.garden', 'brightness': '255' } } ;
        break;

 `hassInput()`:
       
        case 'sensor.garden_brightness':
        document.getElementById('garden_slider').value = state ;
        break;

`<div id="userinput">`:
       
       <button id="garden_toggle">Garden lamp</button>
       <input  id="garden_slider" type="range" min="0" max="255" />
       <button id="garden_full">MAX</button>
       
## Page internal actions 

In `userInput()` there may be elements that has nothing to do with HA. For example:
- disconnect and reconnect 
- hide and show parts of the page
- enter values

In this case, actions must not set the `command` variable.     

# Secuity 

In the case of HA2Web you have to make a trade-off between security and userfrendlyness. 
This is further explained in my document...

### Without access from the internet
This is the simplest case. Only devices on your local network can access web server and MQTT broker. Security is dependent on who has access to your local network, in practise, your wifi. 

### HA web server with access from the internet, but not the MQTT broker 
This is a common case where you, for instance, are using HA Companion to access your system, also when you are not at home. In this case you hide the data traffic using HTTPS protocol. This goes also for the web server traffic. However, the web server is still open so anyone can access your HA2Web pages. Without opening the MQTT broker for access from the internet, you can make a secure HA2Web system. But only accessible from your local network.

### Using ZeroTier  
The ZeroTier add-on for HA gives you a parallel local network that also can be accessed from any client that you have granted access to it. In pracatise you can access it from any of your own computers, mobile phones or tablets. You may let others join your network but that is like giving access to your wifi. 

## Using Nginx
The Nginx Proxy Manager add-on for HA can give you internet access to HA web server and the MQTT broker, and you can protect certain pages using real username/password login.
However, for the MQTT broker you have to stick to the MQTT username/password. 

## MQTT Client is not a user
When a user makes a login, the username/password is stored in a secure way in the browser. The next time, no login is needed.

For the MQTT broker this is not possible. Instead the password has to be entered for each login, provided as query parameters in the URL or as plain text in the page code.

Plain text in the code is not as bad as it sounds. If the page itself requires a user login, no one else can spy into the code.

Password in the query parameters works also for unprotected web pages. However, a password in the URL becomes very visible and may be found in the history logs.

There is also a fourth choice. The browser may store data for you, but in a more insecure way. That is, a user may enter a password only once.   




### HA Cloud




