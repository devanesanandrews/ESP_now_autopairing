How the Code Works
We already explained how the server code works in great detail in a previous project. So, we’ll just take a look at the relevant parts for auto-pairing.

Message Types
The server and senders can exchange two types of messages: messages with pairing data with MAC address, channel, and board id, and messages with the actual data like sensor readings.

So, we create an enumerated type that holds the possible incoming message types (PAIRING and DATA).

enum MessageType {PAIRING, DATA,};
“An enumerated type is a data type (usually user-defined) consisting of a set of named constants called enumerators. The act of creating an enumerated type defines an enumeration. When an identifier such as a variable is declared having an enumerated type, the variable can be assigned any of the enumerators as a value“. Source: https://playground.arduino.cc/Code/Enum/

After that, we create a variable of that type we’ve just created called messageType. Remember that this variable can only have two possible values: PAIRING or DATA.

MessageType messageType;
Data Structure
Create a structure that will contain the data we’ll receive. We called this structure struct_message and it contains the message type (so that we know if we received a message with data or with peer info), board ID, temperature and humidity readings, and the reading ID.

typedef struct struct_message {
  uint8_t msgType;
  uint8_t id;
  float temp;
  float hum;
  unsigned int readingId;
} struct_message;
We also need another structure to contain the peer information for pairing the peer. We call this structure struct_pairing. This structure will contain the message type, board id, mac address of the sender board, and Wi-Fi channel.


typedef struct struct_pairing {       // new structure for pairing
    uint8_t msgType;
    uint8_t id;
    uint8_t macAddr[6];
    uint8_t channel;
} struct_pairing;
We create two variables of type struct_message, one called incomingReadings that will store the readings coming from the slaves, and another called outgoingSetpoints that will hold the data to send to the slaves.

struct_message incomingReadings;
struct_message outgoingSetpoints;
We also create a variable of type struct_pairing to hold the peer information.

struct_pairing pairingData;
readDataToSend() Function
The readDataToSend() should be used to get data from whichever sensor you’re using and put them on the associated structure to be sent to the slave boards.

void readDataToSend() {
  outgoingSetpoints.msgType = DATA;
  outgoingSetpoints.id = 0;
  outgoingSetpoints.temp = random(0, 40);
  outgoingSetpoints.hum = random(0, 100);
  outgoingSetpoints.readingId = counter++;
}
The msgType should be DATA. The id corresponds to the board id (we’re setting the server board ID to 0, the others boards should have id=1, 2, 3, and so on). Finally, temp and hum hold the sensor readings. In this case, we’re setting them to random values. You should replace that with the correct functions to get data from your sensor. Every time we send a new set of readings, we increase the counter variable.

Adding a Peer
We create a function called addPeer() that will return a boolean variable (either true or false) that indicates whether the pairing process was successful or not. This function tries to add peers. It will be called later when the board receives a message of type PAIRING. If the peer is already on the list of peers, it returns true. It also returns true if the peer is successfully added. It returns false, if it fails to add the peer to the list.


bool addPeer(const uint8_t *peer_addr) {      // add pairing
  memset(&slave, 0, sizeof(slave));
  const esp_now_peer_info_t *peer = &slave;
  memcpy(slave.peer_addr, peer_addr, 6);
  
  slave.channel = chan; // pick a channel
  slave.encrypt = 0; // no encryption
  // check if the peer exists
  bool exists = esp_now_is_peer_exist(slave.peer_addr);
  if (exists) {
    // Slave already paired.
    Serial.println("Already Paired");
    return true;
  }
  else {
    esp_err_t addStatus = esp_now_add_peer(peer);
    if (addStatus == ESP_OK) {
      // Pair success
      Serial.println("Pair success");
      return true;
    }
    else 
    {
      Serial.println("Pair failed");
      return false;
    }
  }
} 
Receiving and Handling ESP-NOW Messages
The OnDataRecv() function will be executed when you receive a new ESP-NOW packet.

void OnDataRecv(const uint8_t * mac_addr, const uint8_t *incomingData, int len) {


Inside that function, print the length of the message and the sender’s MAC address:

Serial.print(len);
Serial.print(" bytes of data received from : ");
printMAC(mac_addr);
Previously, we’ve seen that we can receive two types of messages: PAIRING and DATA. So, we must handle the message content differently depending on the type of message. We can get the type of message as follows:

uint8_t type = incomingData[0];       // first message byte is the type of message
Then, we’ll run different codes depending if the message is of type DATA or PAIRING.

If it is of type DATA, copy the information in the incomingData variable into the incomingReadings structure variable.

memcpy(&incomingReadings, incomingData, sizeof(incomingReadings));
Then, create a JSON document with the received information (root):

// create a JSON document with received data and send it by event to the web page
root["id"] = incomingReadings.id;
root["temperature"] = incomingReadings.temp;
root["humidity"] = incomingReadings.hum;
root["readingId"] = String(incomingReadings.readingId);
Convert the JSON document to a string (payload):

serializeJson(root, payload);
After gathering all the received data on the payload variable, send that information to the browser as an event (“new_readings”).

events.send(payload.c_str(), "new_readings", millis());
We’ve seen on a previous project how to handle these events on the client side.

If the message is of type PAIRING, it contains the peer information.

case PAIRING:                            // the message is a pairing request


We save the received data in the incomingData variable and print the details on the Serial Monitor.

memcpy(&pairingData, incomingData, sizeof(pairingData));
Serial.println(pairingData.msgType);
Serial.println(pairingData.id);
Serial.print("Pairing request from: ");
printMAC(mac_addr);
Serial.println();
Serial.println(pairingData.channel);
The server responds back with its MAC address (in access point mode) and channel, so that the peer knows it sent the information using the right channel and can add the server as peer.

if (pairingData.id > 0) {     // do not replay to server itself
  if (pairingData.msgType == PAIRING) { 
    pairingData.id = 0;       // 0 is server
    // Server is in AP_STA mode: peers need to send data to server soft AP MAC address 
    WiFi.softAPmacAddress(pairingData.macAddr);   
    pairingData.channel = chan;
    Serial.println("send response");
    esp_err_t result = esp_now_send(mac_addr, (uint8_t *) &pairingData, sizeof(pairingData));
Finally, the server adds the sender to its peer list using the addPeer() function we created previously.

addPeer(mac_addr);
Initialize ESP-NOW
The initESP_NOW() function intializes ESP-NOW and registers the callback functions for when data is sent and received.

void initESP_NOW(){
    // Init ESP-NOW
    if (esp_now_init() != ESP_OK) {
      Serial.println("Error initializing ESP-NOW");
      return;
    }
    esp_now_register_send_cb(OnDataSent);
    esp_now_register_recv_cb(OnDataRecv);
}


setup()
In the setup(), print the board MAC address:

Serial.println(WiFi.macAddress());
Set the ESP32 receiver as station and soft access point simultaneously:

WiFi.mode(WIFI_AP_STA);
The following lines connect the ESP32 to your local network and print the IP address and the Wi-Fi channel:

WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Setting as a Wi-Fi Station..");
  }
Print the board MAC address in access point mode, which is different than the MAC address on station mode.

Serial.print("Server SOFT AP MAC Address:  ");
Serial.println(WiFi.softAPmacAddress());
Get the board Wi-Fi channel and print it in the Serial Monitor.

chan = WiFi.channel();
Serial.print("Station IP Address: ");
Serial.println(WiFi.localIP());
Serial.print("Wi-Fi Channel: ");
Serial.println(WiFi.channel());
Initialize ESP-NOW by calling the initESP_NOW() function we created previously.

initESP_NOW();
Send Data Messages to the Sender Boards
In the loop(), every 5 seconds (EVENT_INTERVAL_MS) get data from a sensor or sample data by calling the readDataToSend() function. It adds new data to the outgoingSetpoints structure.



readDataToSend();
Finally, send that data to all registered peers.

esp_now_send(NULL, (uint8_t *) &outgoingSetpoints, sizeof(outgoingSetpoints));
That’s pretty much how the server code works when it comes to handling ESP-NOW messages and automatically adding peers.

Testing the Server
After uploading the code to the receiver board, press the on-board EN/RST button. The ESP32 IP address should be printed on the Serial Monitor as well as the Wi-Fi channel.

ESP-NOW Server Serial Monitor Connecting to Wi-fi and printing Wi-Fi channel
You can access the web server on the board’s IP address. At the moment, there won’t be any data displayed because we haven’t prepared the sender boards yet. Let the server board run the code.
