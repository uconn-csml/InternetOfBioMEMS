## Internet of BioMEMS (Biological MicroElectroMechanical System) 

Yong-Jun Shin (yshin@engr.uconn.edu)

Computational and Systems Medicine Lab, Department of Biomedical Engineering, University of Connecticut (http://csml.uconn.edu)



![alt tag](https://cdn.rawgit.com/uconn-csml/InternetOfBioMEMS/master/blockDiagram2.png)

Digital signal processing and control techniques enable the automation of BioMEMS (Biological MicroElectroMechanical System) applications. However, many BioMEMS research labs often underutilize these tools as their implementation requires substantial time, efforts, and expertise. In this context, a cloud platform, which provides digital signal processing and control capabilities  (e.g., the PID control) as real-time web services, can be immensely useful. These cloud-enabled web services will be readily consumed by BioMEMS research labs any time anywhere across the internet. This tutorial is about implementing a cloud-enabled PID controller. For the demonstration purpose, a photoresistor (representing a BioMEMS sensor) and an LED (representing a BioMEMS actuator) are used as these components are readily available. You will need: 

  - Microsoft Azure (cloud) subscription: [Free-trial] [free azure], [Azure Education Grant][azureEdu], [Azure Research Award][azureResearch]
  - One (1) [Arduino Uno][arduinoUno]: A simple microcontroller board and open-source development environment 
  - One (1) Computer (Windows OS): This demo was tested using Windows 7 and 10.  
  - One (1) USB A to USB B cable (for connecting Arduino with Computer)
  - One (1) Photoresistor (GM5539) or any BioMEMS sensor compatible with Arduino
  - One (1) LED or any BioMEMS actuator compatible with Arduino
  - One (1) Resistor (10K ohms)
  - One (1) Protoboard
  - Jumper wires
  - [Arduino Software] [arduinoSoftware]  (Integrated Development Environment)
  - [Node.js] [node]: An open-source, cross-platform runtime environment (uses Javascript)
  - [Johnny-five] [jf]: The Javascript Robotics Programming Framework
  - [socket.io] [socket]: An open-source library enabling persistent, bi-directional, and real-time communication across the internet (supports Javascript and enables communication between index.html (web client) and arduino.js (Node.js)
  - [SignalR] [signalr]: An open-source library enabling persistent, bi-directional, and real-time communication across the internet
(supports Javascript, C#, Java, C++, Objective-C, etc., and enables communication between index.html (web client) and ArduinoHub.cs (Azure)
  - [Visual Studio Community (free)] [vs]: For editing index.html and ArduinoHub.cs (and Startup.cs)
  - [WebStorm (free for students)] [ws]: For editing arduino.js (you may also use Visual Studio)

In this simple demo, a PID controller will be provided as a SignalR web service, which will modulate the photoresistor value by constantly updating the LED brightness to make it close to the reference value. 

### Step 1: Download and install Node.js
[Node.js Downloads] (https://nodejs.org/en/download/) 

### Step 2: Install Johnny-five and socket.io
* Go to your project direcotry using Windows Command Shell (Start > Run > cmd)
* Run: npm install johnny-five (npm is the package manager for node.js)
* Run: npm install socket.io

### Step 3: Setup your Arduino and run following Javascript code
![alt tag](https://cdn.rawgit.com/uconn-csml/InternetOfBioMEMS/master/arduinoSetup.png)
* Downloand and install [Arduino Software] [arduinoSoftware]  (Integrated Development Environment)
* Select serial port (Select Tools > Port > ... )
* Make sure "Standard Firmata" is installed on the Arduino board (Select File > Examples > Firmata > StandardFirmata and Click "Upload")
* Run: node arduino.js

```javascript
// arduino.js
// Yong-Jun Shin (yshin@engr.uconn.edu)
// UCONN Computational and Systems Medicine Lab (csml.uconn.edu), 2016
var five = require("johnny-five"), board = new five.Board();

var io = require('socket.io')(8000);

board.on("ready", function() {

    // Create a new BioMEMS sensor (photoresistor) instance.
    sensor = new five.Sensor({
        pin: "A0", // Arduino analog input pin number
        freq: 500  // Sampling frequency (500 milliseconds = 2 Hz)
    });

    // When a new sensor value is avaiable ("data"), execute {}
    sensor.on("data", function() {
        console.log(this.value); // Show the sensor value on Windows Command Shell
        io.emit('sensorValue', this.value); //Send the sensor value named "sensorValue" to the web client (e.g., sensorValue: 300)
    });

    // Create a new BioMEMS actuator (LED) instance on pin 11
    var actuator = new five.Led(11);
    io.on('connection', function(socket){
        // When the control value is received from the web client, execute {}
        socket.on('control', function(controlValue){
            actuator.brightness(controlValue); //Change the LED brightness to "controlValue"
        });
    });
});
```
### Step 4: Learn how SignalR works!
[Tutorial: Getting Started with SignalR](http://www.asp.net/signalr/overview/getting-started/tutorial-getting-started-with-signalr) 

### Step 5: Create index.html (SignalR web client), which can talk to both Arduino and Azure

``` html
<!DOCTYPE html>
<html>
<head>
    <meta name="author" content="Yong-Jun Shin (yshin@engr.uconn.edu)" />
    <meta name="affiliation" content="UCONN Computational and Systems Medicine Lab (csml.uconn.edu), 2016" />
    <title>Internet of BioMEMS</title>
    <!--Script references. -->
    <script src="https://cdn.socket.io/socket.io-1.3.7.js"></script>
    <script src="http://code.jquery.com/jquery-1.6.4.min.js"></script>
    <script src="http://ajax.aspnetcdn.com/ajax/signalr/jquery.signalr-2.2.0.min.js"></script>
    <!--Reference the autogenerated SignalR hub script. -->
    <script src="signalr/hubs"></script>
</head>
<body>
    Photoresistor value: <div id="sensorValue"></div>
    LED brightness: <div id="controlValue"></div>
    <!--Add script to update the page and send messages.-->
    <script type="text/javascript">
      $(function () {
        var socket = io.connect("http://localhost:8000");
        // Declare a proxy to reference the hub.
        var arduinoHub = $.connection.arduinoHub;
        // Create a function that the hub can call to broadcast messages (data).
        arduinoHub.client.broadcastMessage = function (sensor_value, control_value) {
        // Add the message (data) to the page.
        $('#controlValue').text(control_value);
        $('#sensorValue').text(sensor_value);
            socket.emit('control', control_value); // Send control signal to Arduino
        };

        // Call the Send method on the hub.
        $.connection.hub.start().done(function () {
          // Recieve sensor signal from Arduino
          socket.on('sensorValue', function (value) {
              arduinoHub.server.send(value);
              $('#sensorValue').text(value);
          });
        });
      });
    </script>
  </body>
</html>
```
### Step 6: Implement ArduinoHub.cs (SignalR hub app)
``` cs
// Yong-Jun Shin (yshin@engr.uconn.edu)
// UCONN Computational and Systems Medicine Lab (csml.uconn.edu), 2016
// ArduinoHub.cs
using Microsoft.AspNet.SignalR;
using System;
namespace ArduinoHub
{
    public class ArduinoHub : Hub
    {
        public void Send(string sensor_val)
        {
            // simple P controller
            // control signal for LED should be between 0 and 255
            double Kp = 1; // Kp
            double reference = 600;
            double maxSensor = 1023; // sensor input data range: 0 - 1023
            double sensorValue = Convert.ToDouble(sensor_val);
            double error = reference - sensorValue; //error range: 0 - 1023
            double controlValue = error / maxSensor * Kp * 255; // error/maxSensor range: 0 - 1
            if (controlValue < 0) controlValue = 0;  // lower limit for control signal
            if (controlValue > 255) controlValue = 255; // upper limit for control signal

            // Call the broadcastMessage method to update clients.
            Clients.All.broadcastMessage(sensorValue, controlValue);
        }
    }
}
```
```cs
// Startup.cs
using Microsoft.Owin;
using Owin;
[assembly: OwinStartup(typeof(ArduinoHub.Startup))]
namespace ArduinoHub
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            // Any connection or hub wire up and configuration should go here
            app.MapSignalR();
        }
    }
}
```
### Step 7: Use Google Chrome for testing

[![ScreenShot](https://cdn.rawgit.com/uconn-csml/InternetOfBioMEMS/master/screenshot.png)](https://www.youtube.com/watch?v=sX60sUeqlg4)

[free azure]: <https://azure.microsoft.com/en-us/pricing/free-trial/>
[azureEdu]: <https://azure.microsoft.com/en-us/community/education/>
[azureResearch]: <http://research.microsoft.com/en-us/projects/azure/default.aspx?utm_content=buffer488ef&utm_source=buffer&utm_medium=google&utm_campaign=Buffer>
[arduinoUno]: <https://www.arduino.cc/en/Main/ArduinoBoardUno>
[arduinoSoftware]: <https://www.arduino.cc/en/Main/Software>
[node]: <https://nodejs.org>
[jf]:<http://johnny-five.io/>
[signalr]:<http://www.asp.net/signalr>
[socket]:<http://socket.io/>
[vs]: <https://www.visualstudio.com/en-us/visual-studio-homepage-vs.aspx>
[ws]: <https://www.jetbrains.com/student/>
[signalr tutorial]: <hhttp://www.asp.net/signalr/overview/getting-started/tutorial-getting-started-with-signalr>
