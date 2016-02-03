# InternetOfBioMEMS

This tutorial is about controlling a BioMEMS (Biological MicroMechanical System) usig a cloud-enabled PID controller. You will need:

  - [Microsoft Azure (cloud) account] [free azure]
  - One (1) [Arduino][arduino]: A simple microcontroller board and open-source development environment 
  - [Arduino Software] [arduinoSoftware]  (Integrated Development Environment)
  - [Node.js] [node]: An open-source, cross-platform runtime environment (uses Javascript)
  - [Johnny-five] [jf]: The Javascript Robotics Programming Framework
  - [socket.io] [socket]: An open-source library enabling persistent, bi-directional, and real-time communication across the internet (supports Javascript and enables communication between index.html (web client) and arduino.js (Node.js)
  - [SignalR] [signalr]: An open-source library enabling persistent, bi-directional, and real-time communication across the internet
(supports Javascript, C#, Java, C++, Objective-C, etc., and enables communication between index.html (web client) and ArduinoHub.cs (Azure)
  - [Visual Studio] [vs]: For editing index.html and ArduinoHub.cs (and Startup.cs)
  - [WebStorm] [ws]: For editing arduino.js (you may also use Visual Studio)
  - Computer (Windows OS)
  - One (1) USB A to USB B cable (for connecting Arduino to Computer)
  - One (1) Photoresistor (GM5539) or any BioMEMS sensor compatible with Arduino
  - One (1) LED or any BioMEMS actuator compatible with Arduino
  - One (1) Resistor (10K ohms)
  - One (1) Protoboard
  - Jumper wires

In this simple demo, a PID controller will be provided as a SignalR web service, which will modulate the photoresistor value by constantly updating the LED brightness to make it close to the reference value. 

### Step 1: Download and install Node.js
[Node.js Downloads] (https://nodejs.org/en/download/) 

### Step 2: Install Johnny-five and socket.io
* Go to your project direcotry using Windows Command Shell (Start > Run > cmd)
* Run: npm install johnny-five (npm is the package manager for node.js)
* Run: npm install socket.io

### Step 3: Setup your Arduino and run following Javascript code
* Downloand and install [Arduino Software] [arduinoSoftware]  (Integrated Development Environment)
* Select serial port (Select Tools > Port > ... )
* Make sure "Standard Firmata" is installed on the Arduino board (Select File > Examples > Firmata > StandardFirmata and Click "Upload")
* Run: node arduino.js
* [Using Photoresistors (LDRs) with an Arduino] (https://blog.udemy.com/arduino-ldr/)
* [Photoresistor and LED setup] (http://labalec.fr/erwan/wp-content/uploads/2014/03/LDR_bb.png)
```javascript
// arduino.js
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
    <title>Internet of BioMEMS</title>
    <!--Script references. -->
    <script src="https://cdn.socket.io/socket.io-1.3.7.js"></script>
    <script src="http://code.jquery.com/jquery-1.6.4.min.js"></script>
    <script src="http://ajax.aspnetcdn.com/ajax/signalr/jquery.signalr-2.2.0.min.js"></script>
    <!--Reference the autogenerated SignalR hub script. -->
    <script src="signalr/hubs"></script>
  </head>
  
  <body>
    Photoresistor value: <div id="sensing"></div>
    LED brightness: <div id="control"></div>

    <!--Add script to update the page and send messages.-->
    <script type="text/javascript">
      $(function () {
        var socket = io.connect("http://localhost:8000");
        // Declare a proxy to reference the hub.
        var arduinoHub = $.connection.arduinoHub;
        // Create a function that the hub can call to broadcast messages (data).
        arduinoHub.client.broadcastMessage = function (sensed_value, control_value) {
        // Add the message (data) to the page.
        $('#control').text(control_value);
        $('#sensing').text(sensed_value);
            socket.emit('control', control_value); // Send control signal to Arduino
        };
        
        // Call the Send method on the hub.
        $.connection.hub.start().done(function () {
          // Recieve sensor signal from Arduino
          socket.on('sensing', function (value) {
              arduino.server.send(value);
              $('#sensing').text(value);
          });
        });
      });
    </script>
</body>
</html>
```
### Step 6: Implement ArduinoHub.cs (SignalR hub app)
``` cs
using System;
using System.Web;
using Microsoft.AspNet.SignalR;
namespace ArduinoHub
{
    public class ArduinoHub : Hub
    {
        public void Send(string sensor_val)
        {
            // simple P controller
            // control signal for LED should be between 0 and 255
            doubble Kp = 1; // Kp
            double reference = 600;  
            double maxSensor = 1023; // sensor input data range: 0 - 1023
            double sensorValue = Convert.ToDouble(sensor_val);
            double error = reference - sensorValue; //error range: 0 - 1023
            double controlValue = error/maxSensor * Kp * 255; // error/maxSensor range: 0 - 1
            if (controlValue < 0) controlValue = 0;  // lower limit for control signal
            if (controlValue > 255) controlValue = 255; // upper limit for control signal

            // Call the broadcastMessage method to update clients.
            Clients.All.broadcastMessage(sensorValue, controlValue);
        }
    }
}
```

[free azure]: <https://azure.microsoft.com/en-us/pricing/free-trial/>
[arduino]: <https://www.arduino.cc/>
[arduinoSoftware]: <https://www.arduino.cc/en/Main/Software>
[node]: <https://nodejs.org>
[jf]:<http://johnny-five.io/>
[signalr]:<http://www.asp.net/signalr>
[socket]:<http://socket.io/>
[vs]: <https://www.visualstudio.com/en-us/visual-studio-homepage-vs.aspx>
[ws]: <https://www.jetbrains.com/student/>
[signalr tutorial]: <hhttp://www.asp.net/signalr/overview/getting-started/tutorial-getting-started-with-signalr>
