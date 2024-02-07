# Controller Firmware Overview

The controller firmware can be found [here](https://github.com/0102io/testing/tree/auto-light-sleep/tt_driver_fw_pio).

The controller's primary job is to receive a set of taps via the [bluetooth protocol](./protocol.md) and execute the taps with precise timing. The code is summarized by this flow chart:

![controller firmware flowchart](../images/controllerFirmwareFlowchart.png)

# Compiling the Firmware

This project uses esp32-arduino, with re-compiled libraries made with the [esp32-arduino-lib-builder project](https://github.com/espressif/esp32-arduino-lib-builder), and it is structured for compilation with PlatformIO (and we use the VS Code extension). 

To set these up for this project, follow these steps:
1. Install the Arduino IDE, and then follow the steps in the [arduino-esp32](https://docs.espressif.com/projects/arduino-esp32/en/latest/installing.html).
2. Clone the library builder repo, then switch to the `release/v4.4` branch (we currently use functions that have been removed arduino-esp32 v3.0.0).
3. Edit `esp32-arduino-lib-builder/configs/defconfig.esp32s3` (settings for this project are located in the `documentation` folder of the controller firmware repository).
4. From the library builder directory, run ```./build.sh -t esp32s3```, then ```./tools/copy-to-arduino.sh```.
5. Install VS Code and the PlatformIO extension, as per their guides.
6. Build the `TappyTap-ControllerFirmware` project.
7. Find your .platformio directory (probably at `/Users/[your_username]/.platformio/packages/framework-arduinoespressif32` if you're on a Mac). Then copy these files:
   - `esp32-arduino-lib-builder/out/package_esp32_index.template.json` to `.platformio/packages/framework-arduinoespressif32/package/`
   - `esp32-arduino-lib-builder/out/tools/esp32-arduino-libs/versions.txt` and `esp32-arduino-lib-builder/out/tools/esp32-arduino-libs/esp32s3` to `.platformio/packages/framework-arduinoespressif32/tools/sdk/`.

Next you'll need to move the custom board json files into your platformio folder. From the repository's root directory, copy the files from `./boards` into your platformio boards directory (for example, on a Mac this would likely be in `/Users/[your username]/.platformio/platforms/espressif32/boards`). 

Once you have those set up, you should be able to hit upload and let PlatformIO handle the rest.

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