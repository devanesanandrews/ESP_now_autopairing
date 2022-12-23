How the Code Works  
The ESP32 and ESP8266 are slightly different when it comes to the ESP-NOW-specific functions.   But they are structured similarly. So, we’ll just take a look at the ESP32 code.  

We’ll take a look at the relevant sections that handle auto-pairing with the server. The rest of   the code was already explained in great detail in a previous project.  

Set Board ID  
Define the sender board ID. Each board should have a different id so that the server knows who   sent the message. Board id 0 is reserved for the server, so you should start numbering your   sender boards at 1.  
 
// Set your Board ID (ESP32 Sender #1 = BOARD_ID 1, ESP32 Sender #2 = BOARD_ID 2, etc)  
#define BOARD_ID 1
Define the maximum number of channels
The sender will loop through different Wi-Fi channels until it finds the server. So, set the maximum number of channels.

#define MAX_CHANNEL 11  // for North America // 13 in Europe
Server’s MAC Address
The sender board doesn’t know the server MAC address. So, we’ll start by sending a message to the broadcast MAC address FF:FF:FF:FF:FF:FF on different channels. When we send messages to this MAC address, all ESP-NOW devices receive this message. Then, the server will respond back with its actual MAC address when we find the right Wi-Fi channel.

uint8_t serverAddress[] = {0xFF,0xFF,0xFF,0xFF,0xFF,0xFF};
Data Structure
Similarly to the server code, we create two structures. One to receive actual data and another to receive details for pairing.


//Structure to send data
//Must match the receiver structure
// Structure example to receive data
// Must match the sender structure
typedef struct struct_message {
  uint8_t msgType;
  uint8_t id;
  float temp;
  float hum;
  unsigned int readingId;
} struct_message;

typedef struct struct_pairing {       // new structure for pairing
    uint8_t msgType;
    uint8_t id;
    uint8_t macAddr[6];
    uint8_t channel;
} struct_pairing;

//Create 2 struct_message 
struct_message myData;  // data to send
struct_message inData;  // data received
struct_pairing pairingData;
Pairing Statues
Then, we create an enumeration type called ParingStatus that can have the following values: NOT_PAIRED, PAIR_REQUEST, PAIR_REQUESTED, and PAIR_PAIRED. This will help us following the pairing status situation.

enum PairingStatus {NOT_PAIRED, PAIR_REQUEST, PAIR_REQUESTED, PAIR_PAIRED,};
We create a variable of that type called pairingStatus. When the board first starts, it’s not paired, so it’s set to NOT_PAIRED.

PairingStatus pairingStatus = NOT_PAIRED;
Message Types
As we did in the server, we also create a MessageType so that we know if we received a pairing message or a message with data.



enum MessageType {PAIRING, DATA,};
MessageType messageType;
Adding a Peer
This function adds a new peer to the list. It accepts as arguments the peer MAC address and channel.

void addPeer(const uint8_t * mac_addr, uint8_t chan){
  esp_now_peer_info_t peer;
  ESP_ERROR_CHECK(esp_wifi_set_channel(chan ,WIFI_SECOND_CHAN_NONE));
  esp_now_del_peer(mac_addr);
  memset(&peer, 0, sizeof(esp_now_peer_info_t));
  peer.channel = chan;
  peer.encrypt = false;
  memcpy(peer.peer_addr, mac_addr, sizeof(uint8_t[6]));
  if (esp_now_add_peer(&peer) != ESP_OK){
    Serial.println("Failed to add peer");
    return;
  }
  memcpy(serverAddress, mac_addr, sizeof(uint8_t[6]));
}
Receiving and Handling ESP-NOW Messages
The OnDataRecv() function will be executed when you receive a new ESP-NOW packet.

void OnDataRecv(const uint8_t * mac_addr, const uint8_t *incomingData, int len) {
Inside that function, print the length of the message and the sender’s MAC address:

Serial.print("Packet received from: ");
printMAC(mac_addr);
Serial.println();
Serial.print("data size = ");
Serial.println(sizeof(incomingData));
Previously, we’ve seen that we can receive two types of messages: PAIRING and DATA. So, we must handle the message content differently depending on the type of message. We can get the type of message as follows:



uint8_t type = incomingData[0];       // first message byte is the type of message
Then, we’ll run different codes depending if the message is of type DATA or PAIRING.

If it is of type DATA, copy the information in the incomingData variable into the inData structure variable.

memcpy(&inData, incomingData, sizeof(inData));
Then, we simply print the received data on the Serial Monitor. You can do any other tasks with the received data that might be useful for your project.

Serial.print("ID  = ");
Serial.println(inData.id);
Serial.print("Setpoint temp = ");
Serial.println(inData.temp);
Serial.print("SetPoint humidity = ");
Serial.println(inData.hum);
Serial.print("reading Id  = ");
Serial.println(inData.readingId);
In this case, we blink the built-in LED whenever the reading ID is an odd number, but you can perform any other tasks depending on the received data.

if (incomingReadings.readingId % 2 == 1){
  digitalWrite(LED_BUILTIN, LOW);
} else { 
  digitalWrite(LED_BUILTIN, HIGH);
}
break;
If the message is of type PAIRING, first we check if the received message is from the server and not from another sender board. We know that because the id variable for the server is 0.

case PAIRING:    // we received pairing data from server
  memcpy(&pairingData, incomingData, sizeof(pairingData));
  if (pairingData.id == 0) {              // the message comes from server


Then, we print the MAC address and channel. This information is sent by the server.

Serial.print("Pairing done for ");
printMAC(pairingData.macAddr);
Serial.print(" on channel " );
Serial.print(pairingData.channel);    // channel used by the server
So, now that we know the server details, we can call the addPeer() function and pass as arguments the server MAC address and channel to add the server to the peer list.

addPeer(pairingData.macAddr, pairingData.channel); // add the server  to the peer list 
If the pairing is successful, we change the pairingStatus to PAIR_PAIRED.

pairingStatus = PAIR_PAIRED;             // set the pairing status
Auto Pairing
The autoPairing() function returns the pairing status.

PairingStatus autoPairing(){
We can have different scenarios. If it is of type PAIR_REQUEST, it will set up the ESP-NOW callback functions and send the first message of type PAIRING to the broadcast address on a predefined channel (starting at 1). After that, we change the pairing status to PAIR_REQUESTED (it means we’ve already sent a request).

case PAIR_REQUEST:
    Serial.print("Pairing request on channel "  );
    Serial.println(channel);

// set WiFi channel   
ESP_ERROR_CHECK(esp_wifi_set_channel(channel,  WIFI_SECOND_CHAN_NONE));
if (esp_now_init() != ESP_OK) {
  Serial.println("Error initializing ESP-NOW");
}

// set callback routines
esp_now_register_send_cb(OnDataSent);
esp_now_register_recv_cb(OnDataRecv);
  
// set pairing data to send to the server
pairingData.msgType = PAIRING;
pairingData.id = BOARD_ID;     
pairingData.channel = channel;

// add peer and send request
addPeer(serverAddress, channel);
esp_now_send(serverAddress, (uint8_t *) &pairingData, sizeof(pairingData));
previousMillis = millis();
pairingStatus = PAIR_REQUESTED;


After sending a pairing message, we wait some time to see if we get a message from the server. If we don’t, we try on the next Wi-Fi channel and change the pairingStatus to PAIR_REQUEST again, so that the board sends a new request on a different Wi-Fi channel.

case PAIR_REQUESTED:
// time out to allow receiving response from server
currentMillis = millis();
if(currentMillis - previousMillis > 250) {
  previousMillis = currentMillis;
  // time out expired,  try next channel
  channel ++;
  if (channel > MAX_CHANNEL){
     channel = 1;
  }   
  pairingStatus = PAIR_REQUEST;
}
break;
If the pairingStatus is PAIR_PAIRED, meaning we’re already paired with the server, we don’t need to do anything.

case PAIR_PAIRED:
   // nothing to do here 
break;
Finally, return the pairingStatus.

return pairingStatus;


setup()
In the setup(), set the pairingStatus to PAIR_REQUEST.

pairingStatus = PAIR_REQUEST;
loop()
In the loop(), check if the board is paired with the server before doing anything else.

if (autoPairing() == PAIR_PAIRED) {
This will run the autoPairing() function and handle the auto-pairing with the server. When the board is paired with the sender (PAIR_PAIRED), we can communicate with the server to exchange data with messages of type DATA.

Sending Messages to the Server
In this case, we’re sending arbitrary temperature and humidity values, but you can exchange any other data with the server.

unsigned long currentMillis = millis();
if (currentMillis - previousMillis >= interval) {
  // Save the last time a new reading was published
  previousMillis = currentMillis;
  //Set values to send
  myData.msgType = DATA;
  myData.id = BOARD_ID;
  myData.temp = readDHTTemperature();
  myData.hum = readDHTHumidity();
  myData.readingId = readingId++;
  esp_err_t result = esp_now_send(serverAddress, (uint8_t *) &myData, sizeof(myData));
}
Testing the Sender Boards
Now, you can test the sender boards. We recommend opening a serial communication with the server on another software like PuTTY for example so that you can see what’s going on on the server and sender simultaneously.

After having the server running, you can upload the sender code to the other boards.

After uploading the code, open the Serial Monitor at a baud rate of 115200 and press the RST button so that the board starts running the code
