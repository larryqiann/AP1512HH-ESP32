## Note: This project was originally developed a long time ago and is being released to the public in archive form to document the hardware that has already been produced. Future updates to this project are not guaranteed, and future firmware and hardware development may not be backwards compatible with it. The hardware has not been thoroughly tested and as of this release, does not have firmware to correspond with the hardware yet. The contents of this repository are provided AS-IS where is, without any warranties or guarantees, express or implied. 

It is likely that there will have to be a Gen2 at some point.

# AP1512HH-ESP32
 Coway AP1512HH/AP-1512/AP 1512 Custom Main Board replacement with ESP32

 This is a custom, third party Main PCB (Development Kit) for the Coway AP1512HH air purifier and most likely, the different variants of this device. This project is intended for use with the *non-smart* versions of this device - the AP-1512HHS uses a board that natively supports wireless connectivity and I am unsure what the layout of this is like. This may be useful for developing an integration with Home Assistant or using ESPHome etc, but I have not developed such a firmware yet.

 **Please see the Teardown, along with its disussions, for general information about the device (https://github.com/device-teardowns/coway-ap1512hh)**


 The board is designed to use the same screw holes, dimensions, connectors, LED, and button placement in order to maximize compatibility and allow for drop in retrofit. It operates at entirely low isolated voltage because the power supply in the original unit provides this abstraction. 
 
 **Please note that despite this, please take precautions as the device was not designed with user access to the electronics in mind. Do not plug the unit in while connecting to the serial programming port, and avoid disassembling the power supply section. Use of this board will void the original regulatory approvals of the unit. This board has not been certified or evaluated to any safety, EMC, or other regulatory standards.**

## Getting Started

The easiest way to get started with the design is by reviewing the PDF schematics and layout, provided in the relevant folder. Some images of the board have been provided below as well.

![Top side of the custom main board](/coway_top.jpg)
![Bottom side of the custom main board](/coway_bottom.jpg)

## Hardware Overview

The PROG header on the board follows Espressif's recommended layout for the ESP-PROG interface. Please see the following for details: https://espressif-docs.readthedocs-hosted.com/projects/espressif-esp-iot-solution/en/latest/hw-reference/ESP-Prog_guide.html

The board integrates an ESP32-WROOM module on board. Many of the LEDs and switches are directly connected to GPIO pads of the chip, while a NPIC6C596 shift register is provided to drive the remaining LEDs. This shift register integrates a storage register to allow the LEDs to be updated all at once. An RGB LED is connected directly to the ESP32 to illuminate the dust sensor LED window. On-board 3.3V regulator, door switch detection, and fan PWM output and frequency feedback are implemented, along with level shift for direct interface with the PPD42 dust sensor.

## Safety Considerations

Use of this development board will void the original regulatory approvals of the unit. This board has not been certified or evaluated to any safety, EMC, or other regulatory standards. This board is intended for professional use by engineers or other qualified persons familiar with the safe operation and use of electrical lab and test equipment and safe working practices and is for development purposes only. This documentation, the hardware, and the design are provided for educational, development, and evaluation purposes only and are provided AS-IS with no guarantees or warranty, implied or stated.

Operation of the fan motor outside of its original specifications may cause it to malfunction, overheat, or pose risk of fire or electric shock. 

Operation of the fan outside of its original operating specifications i.e. at a higher than designed RPM may cause the blower wheel to explode or fragment. Some of these units have been documented to do this even on the original hardware and firmware configuration in Amazon reviews. While testing, always keep the blower case closed and the unit affixed to a stable surface and keep eyes, face, and body parts clear of the air outlet. 

Contact with the moving fan, fan motor, or high voltage components inside the case may cause personal injury or death. Never operate the unit with the case open, and never connect the board to a computer or other hardware while the power is plugged in. When disassembling the device, always discharge and verify zero voltage at all high voltage capacitors before touching any electrical components inside. 

## Misc. Hardware Notes

Here are some useful notes and information used for those developing firmware to be used with this development board:

The board mechanical layout is from scanning the original board using a flatbed scanner. The SVG and other info can be found in the Documentation folder. The board has been fit tested and fits great with the buttons and screw holes, as well as LEDs.

### Fan control

Fan control is accomplished through a two part interface:
1) A PWM output. This is, if I remember correctly, not a particularly high frequency interface. From the point of view of the ESP32, a good starting point would be to output 1000Hz. The power board internally uses an optocoupler to turn this PWM signal into an analog voltage with an RC filter for the motor, so it would be best to not pick too high or too low of a frequency here. The signal should not be audible because of this. In the original firmware, the lowest speed appears to be 20% and the highest 80%. **It is not necessarily advisable to use over 80%, as this may have been chosen to keep the DC bus ripple down, to limit the motor current, and/or to prevent overspeed of the fan wheel.** Either MCPWM or LEDC can be used to drive this output.

2) A PFM (pulse frequency modulation) input. This interface produces a square wave with constant 50% duty cycle (only transitions count - could be always positive or negative depending on motor position) to monitor the rotational speed of the motor. The motor will generate 6 pulses per revolution of the shaft (unlike most PC cooling fans which produce 2). Thus, the RPM can be easily determined by monitoring this pin. **It is strongly recommended to implement over-speed protection, locked rotor protection, and over/under load protection in the firmware.Ideally, the values may be measured using an oscilloscope prior to disassembly, and appropriate parameters selected.** I used to have these measurements but they have been lost, but I can recreate them sometime if there is demand. PID is a suitable method to perform motor control here - bearing in mind the limitations on the bandwidth of the feedback signal at low RPM. The MCPWM capture peripheral is well suited to this task.

### Peripheral control

The 15VEN_5V_OUT pin must be enabled for the fan motor to operate. 

The ION_5V_OUT pin toggles the relay that controls the ionizer of the unit

The door switch is connected to a GPIO on the ESP32 only and does not disable any functions in hardware directly

LED and button control is partially by NPIC6C596 power shift register and partially by direct GPIO control. Please refer to the schematic and the datasheet for this shift register to determine the firmware implementation.

### Dust sensor

The dust sensor originally used in the unit is a Shinyei PPD42. It uses the "Low Pulse Occupancy" method to detect the concentration of dust in the air (see https://aqicn.org/sensor/shinyei/). This type of sensor is not particularly accurate. A simple level shift is provided on board to use the sensor - you might consider using another channel of MCPWM Capture to trigger on this pin going low and cumulatively add the time within a certain period where the signal is LOW. As an alternative, the hardware can be modified to use a RC filter, and the value read from the ADC instead. However, this hardware solution has not been provided on the board.

### Connectors

The original connectors on the board are made by Yeonho (SMH series?). These are difficult to find. The closest replacements I found on LCSC are:

https://www.lcsc.com/product-detail/Wire-To-Board-Connector_DEALON-XHB-2AW_C2904999.html (2 pin, needs locking tab lightly shaved with utility knife)

https://www.lcsc.com/product-detail/Wire-To-Board-Connector_XFCN-M2501RF-06P_C492482.html (6 pin, power supply connector, no mods needed)

https://www.lcsc.com/product-detail/Wire-To-Board-Connector_XFCN-M2501RF-05P_C492481.html (5 pin, dust sensor, needs locking tab lightly shaved with utility knife)

## Thank you for reading the documentation! If you have any questions, feel free to open issues! Good luck on your development!

## License

CERN OHL v2 permissive (Please open issue if other is desired)








