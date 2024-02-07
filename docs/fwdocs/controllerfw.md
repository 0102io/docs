# Controller Firmware Overview

The controller firmware can be found [here](https://github.com/0102io/testing/tree/auto-light-sleep/tt_driver_fw_pio).

The controller's primary job is to receive a set of taps via the [bluetooth protocol](./protocol.md) and execute the taps with precise timing. The code is summarized by this flow chart:

![controller firmware flowchart](../images/controllerFirmwareFlowchart.png)

# To-Dos
1. Reduce power consumption. So far we've managed to reduce power by using the modem-sleep config setting, auto light sleep, and disabling the 12v regulator while we're idle. This results in an idle average current draw of around ~50mA, though it is well below that in between BLE connection events and well above that during those events, which happen at least every 30ms. While tapping, with the regulator enabled, the controller itself seems to draw 150-200mA. We're hoping some wizard our there can change some other settings to bring at least the idle current draw much lower. One thing we've seen is that NimBLE might be more power efficient for bluetooth.
2. Bug: really short on duration + off durations cause the controller to crash. e.g. 0.2 on / 0.5 off it crashes; 0.1 on / 0.6 off okay, 0.4 on / 0.0 off okay
3. Change TapQueue to store data in the same format as a TAP_OUT message - we don't need arrays for each setting. TapHandler::tap() has to be updated accordingly.
4. Add repeatCT and repeatDelay settings (ideally make the above change first though)
5. Figure out what we should do with the IMU data! Orientation correction? Wake from deep sleep?
6. IMU interrupts instead of polling by the ESP?
7. Add HBDriver status register error notifications to the status message (see TapHandler::checkDiagnosticReg)
8. Should we have some kind of auth for connecting to the controller for either normal operation or OTA uploads?
9. Do something with the touch detection pins (e.g. FPC not connected notification)
10. Make it so that the device type (e.g. glove, patch) can be changed through bluetooth.
11. Bring back repeat tap setting.