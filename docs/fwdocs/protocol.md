# Communication Protocol for Client Device and TappyTap Controller

This section outlines the communication protocol for a central device to connect to a the TappyTap Controller using Bluetooth 5 (LE). see 'script.js' in the examples folder for an implementation of this protocol.

## Connecting to the Controller

The Controller advertises itself as a BLE server with the service UUID `4fafc201-1fb5-459e-8fcc-c5c9c331914b`. The client device should initiate a scan for BLE devices and look for the one advertising this service UUID. Once found, the client device should establish a connection to the BLE server (the Controller).

## Sending Messages to the Controller

To send a message from the client device to the Controller, the central device should write to the BLE characteristic with the UUID `beb5483e-36e1-4688-b7f5-ea07361b26a8`.

### Sent Message Types and Structure

- `TAP_OUT`: Byte code `1`. The message should follow this structure:

    - Byte 0: Message type (`1`)
    - Byte 1: idenfication number that is retruned by the controller along with the confirmation when the TAP_OUT sequence is completed. Sending followup TAP_OUT messages will have their contents added to the queue of taps, and the new ID will overwrite the last. Sending an ID of `0` will cancel an active pattern.
    - Bytes 2 to n: sets of 6 bytes for each tap in the pattern. The 6 bytes are: 
      | Byte | Description | 
      |---|---|
      | 0-1 | onDuration (MSB first). This is length of time to apply voltage to a coil, in tens of microseconds (i.e. onDuration = 20 --> 200us) |
      | 2-3 | offDuration (MSB first). This is the length of time before the next tap executes, in hundreds of microseconds (i.e. offDuration = 20 --> 2000us) |
      | 4 | Anode ID and output pin. Bits 0-3 are the output pin. Bits 4-5 are the anode ID. Bits 6-7 are unused. Bit 0 is the LSb. |
      | 5 | Cathode ID and output pin. Bits 0-3 are the output pin. Bits 4-5 are the anode ID. Bits 6-7 are unused. Bit 0 is the LSb. |

      TAP_OUT example:
      | Byte Value | Description |
      |---|---|
      | 0b00000001 | Message type (TAP_OUT)|
      | 0b00000002 | TapoutID = 2 |
      | 0b00000000 | onDuration MSB = 0 |
      | 0b01100100 | onDuration LSB = 100, i.e. 1000us |
      | 0b00000000 | offDuration MSB = 0 |
      | 0b01100100 | offDuration LSB = 100, i.e. 10000us |
      | 0b--010101 | Andode: ID = 1, output pin = 5 |
      | 0b--101001 | Andode: ID = 3, output pin = 9 |
      

- `GET_DEVICE_INFO`: Byte code `2`. Controller returns with `DEVICE_INFO` message.

- `CANCEL_AND_TAP`: Byte code `3`. Clears the queue of taps and then loads the new taps through the same function called by TAP_OUT. This can be used to clear the queue and start a new pattern in 1 BLE pulse.

- `UPDATE_STATUS_FREQUENCY`: Byte code `4`. A second byte with the frequency in Hz must be sent. If the requested frequency is above the limit set in the firmware, the maxiumum frequency will be used instead. If 0 is sent, the frequency will be set to 1 Hz.

- `CHANGE_DEFAULT_SUBSTRATE`: Byte code `5`. Send with 3 more bytes for `substrateType` (1 = patch, 2 = palm/glove), `substrateVMajor`, and `substrateVMinor`. These values are saved to non-volatile memory and used to set the bluetooth device name. They can be read using `GET_DEVICE_INFO`.

## Receiving Messages from the Controller

The client device should monitor the characteristic with UUID `d036e381-fd38-4376-801f-f5d90ba2ca64` for notifications from the Controller.

### Received Message Types and Structure

- `STATUS_UPDATE`: Byte code `1`. This message is sent to the client device at a user settable frequency. The returned bytes are as follows; unless otherwise noted, multi-byte values are all MSB first


    | Byte | Description |
    |---|---|
    | 0 | Message code (STATUS_UPDATE) |
    | 1 | Battery percent |
    | 2 | Last received tapout id |
    | 3-4 | Headroom / free space in the queue |
    | 5-8 | IMU accel X (multiplied by 1000) |
    | 9-12 | IMU accel Y (multiplied by 1000) |
    | 13-16 | IMU accel Z (multiplied by 1000) |
    | 17-20 | IMU gyro X (multiplied by 1000) |
    | 21-24 | IMU gyro Y (multiplied by 1000) |
    | 25-28 | IMU gyro Z (multiplied by 1000) |
    | 29-30 | IMU temperature (multiplied by 10) |
    <!-- note: when reading from the status update function, remember that an additional byte is added (the message code) -->

- `WARNING`: Byte code `2`. This message contains warnings, and it is sent when they occur (i.e not regularly like the status updates). This message may contain multiple error codes each with data of different lengths. The codes and types of data are as follows:
  - `INVALID_MSG_TYPE`: Byte code `51`, + 1 data byte. The byte code is followed by the 1-byte code of the invalid message type that was received. 
  - `INCORRECT_MSG_SIZE`: Byte code `52`, + 0 data bytes. This warning is sent when the last received message was incorrectly sized (e.g. a TAP_OUT message might be missing a setting or col index, etc.).
  - `PARAM_OOB`: Byte code `53`, + 2 data bytes. The byte code is followed by 2 bytes that are the MSB and LSB of the out of bounds parameter from the last `TAP_OUT` message that was last received. If multiple parameters are OOB, only the first 10 are sent as warnings. Row and column indicies that are OOB are replaced with "empty tap" indicies, which will make the controller pause for the onDuration to keep the pattern cadence intact. If the warning is given for an onDuration parameter (i.e. above the limit), the controller replaces the sent parameter with the maximum onDuration value defined in the firmware. Note that the returned index does not count the message type or tapout id bytes.
  - `OTA_TIMEOUT`: Byte code `61`, + 0 data bytes. This warning is sent when the controller receives an OTA upload request but the user doesn't press the 'interact button' before the timeout period.
  - `QUEUE_FULL`: Byte code `62`, + 2 data bytes. This warning is sent when the last TAP_OUT message contained more taps than could be fit into the queue. The byte code is followed by 2 bytes for the MSB and LSB of the index of the first tap that didn't fit into the queue.
  - `BOARD_OVERHEAT`: Byte code `63`, + 1 data byte. This warning is sent when the board temperature changes to a different "level", with level 0 being normal and higher levels being increasingly hot. For each level above 0, the controller attenuates the tap onDuration while keeping the pattern cadence the same. The level is included as a single byte after the warning code.
  - `OVERTAP`: Byte code `64`, + 2 data bytes. This warning is sent when an individual tapper is being driven at too high of a duty cycle. The following 2 bytes are the tapper's row and column indices. Similar to `BOARD_OVERHEAT`, the controller attenuates the onDuration of the taps on an over-tapped actuator, while keeping the cadence the same (by pausing without tapping).
  - `HBRIDGE_DIAGNOSTIC_ERRORS`: Byte code `65`, + 2 data bytes. This warning is sent when an H-bridge driver chip reports a warning while it is being turned on or off for a tap. The first additional byte of data is the ID of the H-bridge chip that reported errors. The second data byte is a set of warning/error flags listed below. These warnings are described in the driver's [datasheet](https://www.monolithicpower.com/en/documentview/productdocument/index/version/2/document_type/Datasheet/lang/en/sku/MP6527GF/document_id/10142/). A warning message is only sent once when the error byte has changed.
  
    x (MSB) | x | OC_7TO10_ERROR | OC_1TO6_ERROR | OLD_7TO10_ERROR | OLD_1TO6_ERROR | PSF_ERROR | TW_WARNING (LSB)


  - `HBRIDGE_DISABLED`: Byte code `66`, + 0 data bytes. This warning is sent when an H-bridge disbaled event is detected. This can happen when either the overcurrent or watchdog protection circuits pull the H-bridge enable pin low, which disables the H-bridge outputs until they are written to again by the microcontroller. The overcurrent circuit trips if an output has been on for too long (200-400ms). The watchdog timer trips if the microcontroller hasn't pinged it within its timeout period.

- `DEVICE_INFO`: Byte code `3`. This message contains configuration information that usually only needs to be passed to the client device once. The returned bytes are as follows; unless otherwise noted, multi-byte values are all MSB first

    | Byte | Description |
    |---|---|
    | 0 | Message code (DEVICE_INFO) |
    | 1 | Serial Number (MAC Address) first octet |
    | 2 | Serial Number (MAC Address) second octet |
    | 3 | Serial Number (MAC Address) third octet |
    | 4 | Serial Number (MAC Address) fourth octet |
    | 5 | Serial Number (MAC Address) fifth octet |
    | 6 | Serial Number (MAC Address) sixth octet |
    | 7 | Controller hardware version (major) |
    | 8 | Controller hardware version (minor) |
    | 9 | Firmware version (major) |
    | 10 | Firmware version (minor) |
    | 11 | Firmware version (patch) |
    | 12 | Connected board type (0 for swatch, 1 for palm) |
    | 13 | Connected board version (major) |
    | 14 | Connected board version (minor) |
    | 15-16 | Maximum tap on duration (hundredths of a ms) |