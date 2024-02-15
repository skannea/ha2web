# Home Assistant goes to Web - ha2web

This is an instruction on how to build **web applications based on entities that are managed by your Home Assistant** (HA) installation.

**This is not a complete development environment** for web applications. 

This is only 
- a few simple source code files that are easy to understand and modify for your needs
- some ideas about security

# Prerequisites
- An **MQTT broker** (for example, the Mosquitto add-on) must be installed and set up for communication between HA and clients (browsers) running the application.
- The **HA web server** folder */config/www* must exist. Application web pages are placed there.
- The **python_script** functionality must be enabled in *configuration.yaml* (by adding `python_script:` ).
- **Security** aspects must be considered if your HA installation can be accessed from the internet. 

# How it works
## HA automation
- A HA automation (based on the HA2Web blueprint) is set up.
- It defines a *list of entities* that are involved in the web application.
- It triggers when the state changes for one of the entities in the *list of entities* and publishes an MQTT message about this.
- It triggers on incoming MQTT messages that can be translated into entity service calls (such as *light.turn_on*).
- The service is called only if the entity matches the *list of entities*.
  
## Application web page
- The application web page contains code for an MQTT client.
- When the application web page is loaded, it starts subscribing for messages about state changes.
- When a message is received, it is possible to act on it, for example changing color of an icon or displaying a value.
- The application web page contains code that acts on user input, such as button clicks.
- The code makes MQTT client publish MQTT messages that can be translated into entity service calls.


## Multiple clients
The HA automation acts as a server that may serve multiple clients. Each client has loaded the application web page and started the subscription for MQTT messages. A state change will affect all client.

The messages from the HA automation are stored by the MQTT broker. When a client restarts or a new client starts, it will get the states of all entities.  


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

## ha2web_minimal.html
`/config/www/ha2web_minimal.html`

An example HTML file that is working but firstly intended for understanding the code. It contains the required JavaScript code and a very basic example of HTML code. 

The file has MQTT credentials in the code and must not be used if your MQTT broker is exposed to the internet. 

## ha2web_example.html
`/config/www/ha2web_example.html`

An example that is intended to be used from the internet. It can be used as a template or as a start point for your own files. 

The MQTT credentials are provided via URL request parameters, like this: 

`https://pages.mydomain.duckdns.org/ha2web_example.html?broker=mqtt.mydomain.duckdns.org&username=ha2web&password=ha2webpassword`

The code also includes basic logging. 

# HA2web page code
Note the code examples in this section differ from what is in the example files.

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

For example: 
- when a button is clicked, publish a message that makes a *light.toggle* 
- when the state change is received, change the color of the button

See also the example below.
 
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
        let lev = document.getElementById('garden_slider').value;  
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

# Some hints

## Copy from the automation
If you start your work-flow by making an automation (based on the bluescript), you can select *Edit in YAML* and get the complete list of entry_ids. Copy them into your *hassInput* function. 

## Make a debug page
Logging of events and data is very useful but you don't want it on the page. To make a dedicated debug page:
- Make a `<div id="page">` for the application page.
- Make a `<div id="debug">` for the debug page.
- Put a `<div id="log">` in the debug page.
- Add buttons in the debug page, for example to disconnect or restart.
- Add a button in the application page to change page. 

        // when show is true, show debug page 
        // when show is false, show application page  
        function showDebugPage(show) {
            document.getElementById('page'  ).style.display = (show ? 'none' : 'block') ;  
            document.getElementById('debug').style.display = (show ? 'block' : 'none') ;  
        }

        // - log to page
        function log( msg ) { 
          document.getElementById('log').innerHTML += msg + '<br>';
        } 

        ...

      <div id="page">
        <!-- application page -->
        ...
        <button onclick='showDebugPage(true);'     >Go to debug page</button>
      </div>
      <div id="debug" >
        <h2>Debug page</h2>
        <button onclick='showDebugPage(false);'    >Continue</button>
        <button onclick='start();'                 >Restart</button>
        <button onclick='location.reload();'       >Reload page</button>
        <button onclick='mqttclient.disconnect();' >Disconnect</button>
      
        <div id="log"><h2>Log output</h2></div>
      </div>

## Use icons 
- In the `<head>` element, include icon css files:
      
      <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@mdi/font@6.9.96/css/materialdesignicons.min.css"/>
      <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css"/>

- Put icons in `<div id=userinput>`:

      The lamp <i id="thelamp" class="fa fa-bulb" style="font-size:48px;"></i>
  
- Use icons as buttons in `userInput()`:
  
      case 'thelamp': 
      command = { "domain":"light", "service":"toggle", "input":{'entity_id':'light.thelamp' }};
      break;

- Change icon on state changes in `hassInput()`:
  
     case 'light.thelamp':
     document.getElementById('thelamp').style.color = (state == 'on') ? 'gold' : 'grey';
     break;

- Or replace the icon:

     case 'light.thelamp':
     var s =                '<i id="thelamp" class="fa fa-toggle-off" style="font-size:48px; color:grey;"></i>'; 
     if (state == 'on') s = '<i id="thelamp" class="fa fa-toggle-on"  style="font-size:48px; color:gold;"></i>';
     document.getElementById('thelamp').outerHTML = s;
     break;

# Security 

## Without access from the internet
This is the simplest case. Only devices on your local network can access web server and MQTT broker. Security is dependent on who has access to your local network.   

Internet access to HA is required when using HA Companion to access your system, also when you are not at home. But then you also give internet access to your HA2Web pages in *config/www* folder. 

Without opening the MQTT broker for access from the internet, anyone can access your HA2Web pages but not communicate through MQTT. That may only be done when you have access to your local network.

## Using ZeroTier  
The ZeroTier add-on for HA gives you a network that looks like a local network. Any ZeroTier enabled client (PC, mobile phone or tablet) can join the network. You may let others join your network but that is like giving access to your wifi. 

The disadvantages with this solution is that
- it is pretty power consuming so in practise it works best for mains connected devices
- you can't give access to your HA2Web pages without giving access to also other part of your system

## Using Nginx
There are several ways to provide secure connections from the internet to your HA and to your MQTT broker. The *Nginx Proxy Manager* add-on for HA  is a good choice. With some tricks it is even possible to protect certain pages using username/password login. 
[Read more.](https://github.com/skannea/HA-findings/blob/main/Add%20HTTPS%20and%20Login.md)

## Self-stored solution 
In this solution, all parts are in HA:
- use *Nginx Proxy Manager* to secure connections  
- place your HA2Web pages in www folder and let them be unprotected so anyone can access them 
- do not put any MQTT credentials in the page code
- provide the MQTT credentials as query parameters in the URL like
  
`https://ha2web.mydomain.duckdns.org/local/myha2webapp.html?user=ha2webapp&pass=mysecretpassword`

Anybody with the complete URL can access your HA2Web page and establish the MQTT connection. 

## Google Sites solution 
To make it even more secure (and perhaps more useful) you can create a *Google Sites* site with restricted access. In a site page, insert an *iframe* with the URL. This way, the URL will be hidden. 

Google Sites allows you to insert not just the URL but the complete HTML code. Doing so you may put MQTT credentials in the code. Note that you can't edit the code in Google Sites, only paste it. So you have to keep a copy of the code somewhere. 

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







