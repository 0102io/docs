# Communication Protocol for Client Device and TappyTap Driver Board

This section outlines the communication protocol for a client device to connect to a the TappyTap Driver Board using Bluetooth 5 (LE). see 'script.js' in the examples folder for an implementation of this protocol.

## Connecting to the Driver Board

The Driver Board advertises itself as a BLE server with the service UUID `4fafc201-1fb5-459e-8fcc-c5c9c331914b`. The client device should initiate a scan for BLE devices and look for the one advertising this service UUID. Once found, the client device should establish a connection to the BLE server (the Driver Board).

## Sending Messages to the Driver Board

To send a message from the client device to the Driver Board, the client device should write to the BLE characteristic with the UUID `beb5483e-36e1-4688-b7f5-ea07361b26a8`.

### Sent Message Types and Structure

- `TAP_OUT`: Byte code `1`. The message should follow this structure:

    - Byte 0: `1`
    - Byte 1: idenfication number that is published along with the confirmation when the TAP_OUT sequence is completed
    - Byte 2 to n: row and column index pairs for the tap sequences. Each pair consists of the row index followed by the column index. For example, the message 0x01 0xFF 0x00 0x00 0x01 0x01 = [TAP_OUT] [Identifier (FF)] [ROW 0] [COL 0] [ROW 1] [COL 1]. Row/col pairs will be tapped out according to the parameters set with TAP_CONFIG, with the pair after the TAP_OUT byte tapped first.
        - if there are no bytes after the identifier, the current TAP_OUT sequence will be cancelled.

- `TAP_CONFIG`: Byte code `2`. The message should follow this structure:

    - Byte 0: `2`
    - Byte 1: `onDur` (in tenths of a ms, i.e. hundreds of microseconds); range is 1-30
    - Byte 2: `offDur` (in ms); range is 10-255
    - Byte 3: `repeatDelay` (in ms); range is 2-255
    - Byte 4: `repeatCT` (number of repetitions; 1 = tap once); range is 1-5

- `CHECK_BATTERY`: Byte code `3`.

- `TAP_OUT_EXPLICIT`: Byte code `4`. The message should follow this structure:

    - Byte 0: `4`
    - Byte 1: idenfication number that is published along with the confirmation when the TAP_OUT sequence is completed
    - Byte 2 to n: row and column index pairs for the tap sequences, as well as on duration and off duration for each tap. Data is ordered as: [row] [col] [onDuration] [offDuration]. For example, the message 0x04 0xFF 0x00 0x00 0x0A 0x19 0x01 0x01 0x14 0x32 = [TAP_OUT] [Identifier (FF)] [ROW 0] [COL 0] [ON DURATION 1ms] [OFF DURATION 25ms] [ROW 1] [COL 1] [ON DURATION 2ms] [OFF DURATION 50ms]. Currently this will cause repeatCT to be set to 1 (i.e. taps are not repeated). Row/col pairs will be tapped out with the pair after the TAP_OUT byte tapped first. Note that on duration values are converted into hundreds of microseconds, e.g. 0x01 = 100us = 0.1ms.
        - if there are no bytes after the identifier, the current TAP_OUT sequence will be cancelled.

- `GET_VERSION`: Byte code `5`.

- `START_IMU_STREAM`: Byte code `6`.

- `STOP_IMU_STREAM`: Byte code `7`.

- 'TAP_ALL': Byte code '8'. The message should follow this structure:

    - Byte 0: `4`
    - Byte 1: idenfication number that is published along with the confirmation when the TAP_OUT sequence is completed
    - Byte 2: on duration (tenths of a millisecond) that will be used for all taps in the pattern; follows the same limits as TAP_OUT and TAP_OUT_EXPLICIT
    - Byte 3: off duration (tenths of a milliseond - note that this is different from TAP_OUT and TAP_OUT_EXPLICIT which use ms for off duration) that will be used for all taps in the pattern. The lower limit is 0 (no delay), the upper limit is the same as for TAP_OUT and TAP_OUT_EXPLICIT.
    - Byte 4 to n: row and column index pairs for the tap sequences

## Receiving Messages from the Driver Board

The client device should also monitor the same BLE characteristic (UUID `beb5483e-36e1-4688-b7f5-ea07361b26a8`) for notifications to receive messages from the Driver Board.

### Received Message Types and Structure

- `CONFRIM_RECEIVED`: Byte code `1`. This message is sent to the client device to acknowledge that the sent message has been received and processed. This is only sent if the message was error free; otherwise an error code is sent instead.

- `TAPOUT_COMPLETE`: Byte code `2`. This message contains 2 bytes; the first is the byte code and the second is the identifying byte sent to the microcontroller along with the TAP_OUT command. This message is sent to the client device to indicate that the tap out sequence has been completed.

- `BATTERY_PERCENT`: Byte code `3`. This message contains 2 bytes; the first is the byte code and the second is the estimated remaining percentage of charge remaining in the battery pack.

- `GET_VERSION`: Byte code `4`. This message contains 6 bytes; the first is the byte code, then in order: hardware version number major, hardware version number minor, firmware version number major, firmware version number minor, firmware version patch. Version numbers can be read as: Hardware Version [major].[minor], Firmware Version [major].[minor].[patch].

- `IMU_DATA`: Byte code `5`. This message contains 25 bytes; the first is the byte code and the next 24 are the 6 data values polled from the IMU (acceleration x y z and gyro x y z in that order). After collecting the data as floats, the controller multiplies them by 1000 and converts them into int32 values (preserving 3 decimal places), then splits them into 4 bytes each (MSB first). 

- `INVALID_MSG_TYPE`: Byte code `51`. This message contains 2 bytes; the first is the byte code and the second is the invalid message tpye that was received

- `INCORRECT_MSG_SIZE`: Byte code `52`. This message contains 2 bytes; the first is the byte code and the second is the size of the received message, including the sent byte code

- `PARAM_OOB`: Byte code `53`. This message contains 2 bytes; the first is the byte code and the second is the position of the message parameter that is out of bounds. If multiple parameters are OOB, the position of the last one will be sent. For TAP_CONFIG messages, the last received parameter will stay the same (or it will be the default parameter value if no valid config message has been received yet), and a flag is raised to indicate that the last TAP_CONFIG received was invalid (CONFIG_INVALID, see below). This flag is sent each time a new TAP_OUT message is received until a valid TAP_CONFIG message is received
For TAP_ALL messages, invalid parameters are replaced with hard coded default values.

- `CONFIG_INVALID`: Byte code `54`. This message is sent to the client device when a TAP_OUT message is received but the last TAP_CONFIG message had an error in it.

- `ROW_INDEX_OOB`: Byte code `55`. This message contains 2 bytes; the first is the byte code and the second is the position of the out of bounds row index. When an out of bounds index is received, the firmware will not tap an actuator, but it will still use the timing of a normal tap, and continue with the rest of the in bounds taps. If multiple indicies are out of bounds, only the position of the last one will be sent. If a column with a larger index is also OOB, it will be reported instead.

- `COL_INDEX_OOB`: Byte code `56`. This message contains 2 bytes; the first is the byte code and the second is the position of the out of bounds column index. When an out of bounds index is received, the firmware will not tap an actuator, but it will still use the timing of a normal tap, and continue with the rest of the in bounds taps. If multiple indicies are out of bounds, only the position of the last one will be sent. If a row with the same or larger index is also OOB, it will be reported instead.

- `OVERTAP_WARNING`: Byte code `57`. This message is sent to the client device when the TAP_CONFIG parameters have set the combination of onDur, repeatCT, and repeatDelay too high. onDur is automatically reduced to a safer value.

- `DAMAGED_TAPPER`: Byte code `58`. This message contains 3 bytes; the first is the byte code, the second is the row index and the third is the column index. This message is sent when the current measured for a tap doesn't exceed the minimum value considered normal/healthy. 

- `OC_EVENT`: Byte code `59`. This message is sent when an overcurrent event is detected, i.e. when the on board analog circuitry detects a sustained current being drawn by the H-bridge driver ICs. The on board circuitry automatically disables the H bridge outputs. This should only happen in an event where the controller turns an output on and then somehow misses turning it off (e.g. if an SPI timing error causes a write commend to be missed) because the tap configuration settings should prevent a user from entering tap on-durations from reaching this threshold. In cases where the controller crashes mid-tap, this message will likely not be generated.

- `BATTERY_POLL_COOLDOWN`: Byte code `60`. This message is sent to the client device when the controller receives a battery poll request too quickly after a perviously sent request.

- `OTA_TIMEOUT`: Byte code `61`. This message is sent to the client device when the controller receives an OTA upload request but the user doesn't press the 'user button' before the timeout period.