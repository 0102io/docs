# Compiling the Firmware

This project uses re-compiled arduino-esp32 core libraries using the [esp32-arduino-lib-builder project](https://github.com/espressif/esp32-arduino-lib-builder), and it is structured for compilation with PlatformIO. I'm not an expert with PlatformIO, ESP-IDF, the library builder, etc. so I'd recommend following other guides to get those all set up first.

## Steps for Building New Core Libraries

1. Edit `esp32-arduino-lib-builder/configs/defconfig.esp32s3` (settings for this project are located in the documentation folder)

2. In a terminal window, navigate to the `esp32-arduino-lib-builder` folder and enter:
    ```bash
    rm -rf build # remove the existing build files if you've already compiled them
    rm -rf components/esp-rainmaker # I had to do this to get my build on Oct 2 to work, may not be necessary now
    ./build.sh -t esp32s3 -I idf-release/v5.1
    ./tool/copy-to-arduino # if using arduino, or copy the files listed in the script to the relevant location for platformIO - I believe it is /Users/[your_username]/.platformio/packages/framework-arduinoespressif32
    ```
Note regarding the rainmaker module: I last built core files on Oct. 2, 2023, and encountered an error similar to [this GitHub issue](https://github.com/espressif/esp32-arduino-lib-builder/issues/138). For me, pulling the latest update (even fully re-installing the repository) didn't resolve the issue, so I added a specific line to the `update-components.sh` file and removed the existing rainmaker component with the command above.
```bash
#
# CLONE/UPDATE ESP-RAINMAKER
#
echo "Updating ESP-RainMaker..."
if [ ! -d "$AR_COMPS/esp-rainmaker" ]; then
    git clone $RMAKER_REPO_URL "$AR_COMPS/esp-rainmaker" && \
    git -C "$AR_COMPS/esp-rainmaker" checkout 0414a8530ec1ac8714269302503c71c238b68836 # <------ this line
    git -C "$AR_COMPS/esp-rainmaker" submodule update --init --recursive
else
    git -C "$AR_COMPS/esp-rainmaker" fetch && \
    git -C "$AR_COMPS/esp-rainmaker" pull --ff-only && \
    git -C "$AR_COMPS/esp-rainmaker" submodule update --init --recursive
fi
if [ $? -ne 0 ]; then exit 1; fi
```