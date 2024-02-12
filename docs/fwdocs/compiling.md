## Building the firmware:

- Download or build arduino libs using the instructions below to get an `out` folder.
- Install VSCode and the PlatformIO extension
- Build the project once (PlatformIO > Build)
- Find your .platformio directory (probably at `~/.platformio/`). Then copy these files:
  - `out/package_esp32_index.template.json` to `~/.platformio/packages/framework-arduinoespressif32/package/` (the package directory isn't built by platformio by default)
  - `out/tools/esp32-arduino-libs/versions.txt` and `out/tools/esp32-arduino-libs/esp32s3` to `~/.platformio/packages/framework-arduinoespressif32/tools/sdk/.`
- Build the project again

Next you'll need to move the custom board json files into your platformio folder. From the repository's root directory, copy the files from `./boards` into your platformio boards directory (for example, on a Mac this would likely be in `/Users/[your username]/.platformio/platforms/espressif32/boards`). 

Once you have those set up, you should be able to hit upload and let PlatformIO handle the rest.

## Download pre-built arduino libs

Download `out.zip` from https://github.com/0102io/esp32-libs/releases/ and extract it.

## Build arduino libs yourself

> [!NOTE]
> This takes a long time and is error-prone. If possible, use the pre-built files instead.

Create a blank github codespace on Ubuntu 20.04.6 LTS

Run the command:

```
sudo apt-get update &&
    sudo apt-get install -y libusb-1.0.0-dev git wget curl libssl-dev libncurses-dev flex bison gperf python python-setuptools python-cryptography python-pyparsing python-pyelftools cmake ninja-build ccache jq &&
    sudo pip install --upgrade pip &&
    git clone https://github.com/espressif/esp32-arduino-lib-builder &&
    git clone https://github.com/0102io/TappyTap-ControllerFirmware &&
    cp TappyTap-ControllerFirmware/documentation/defconfig.esp32s3 esp32-arduino-lib-builder/configs/defconfig.esp32s3 &&
    cd esp32-arduino-lib-builder && 
    git checkout release/v4.4 && 
    ./build.sh -t esp32s3 &&
    zip -FSr out.zip out
```

Download out.zip from esp32-arduino-lib-builder and extract it.