# Design notes for schematic and PCB for signal generator



## Specifications

### Microcontroller based, purely software function generator

* 10Vpp into 50Ω load
* 100khz sine wave bandwidth
* All packages able to be hand soldered using an iron only (as much as possible, 0805 size was chosen for easy hand soldering)
* As much as possible, use lower cost/jellybean components
* Fully controllable via USB



## Power

The use of USB, barrel jack, and mains input was considered. Mains input was discounted due to safety as I did not want to deal with mains voltage safety as well as the size required for the converter. 1 USB port already populated for communication with PC means USB 2.0 default power available ([5v 500mA](https://en.wikipedia.org/wiki/USB_hardware#Power)), but may not be enough. USB PD is also a possibility, but most laptop USB ports cannot provide USB PD, which means needs a second USB port just for power. To reduce confusion about which port is power and which port is data, a barrel jack was opted for. 10Vpp into 50Ω load is 20Vpp into Hi-Z load. The op-amps then require > ±10V supply for enough headroom to prevent clipping as even rail to rail op amps require few mV headroom. ±12V was decided on as the op amp supply voltages.

Common barrel jack voltages are 5V, 9V, 12V and 24V. When using 5V, 9V and 12V, a boost converter is required for the analog ±12V supply rails as I want to use LDO regulators to reduce noise on the power rails, which require \~13V to output 12V. Thus 24V in the barrel jack is stepped down to +13V for and +4v using buck converters, and -13V is generated using a buck converter IC configured as a inverting buck-boost converter.

The ±13V is stepped down by positive and negative LDOs to reduce the amount of noise in the analog supply rails. The STM32 requires 3.3V on its VDD and VDDA pins, with a choice to either separate or tie together the pins. I wanted a choice on whether to use the internal or external DAC, and the internal DAC requires a noise free VDDA pin. Therefore I decided to use 2 separate LDOs to step down the 4v into 3.3V and 3.3VA rails.

### Input protection

Between the barrel jack and the buck converters, there is a fuse for overcurrent protection and a p-channel mosfet reverse polarity protection. ESD protection was deemed unnecessary due to the large input capacitors.



### Component selection

#### Fuse and Reverse Voltage Protection

A common glass fuse with current rating of XXmA was selected. A PTC fuse was also considered but rejected due to the slow trip time.

Reverse voltage protection using diodes was considered but eventually went with a p-channel mosfet design to minimise waste heat. A common p-channel mosfet, the [AO3401A](https://www.aosmd.com/sites/default/files/res/datasheets/AO3401A.pdf) was selected due to its low cost, high availability. The gate clamping diode [BZT52C10](https://www.diodes.com/datasheet/download/BZT52Cxx.pdf) was selected due to there being multiple manufacturers while being cheap.

#### Buck converters

Initially to reduce BOM count, 3 identical buck converters were used. However, the high current required by the MCU is not required for the ±12V supply rails, and higher current converters and their associated components are much more expensive. Thus, a high current buck converter was used for the +3.3V rails and a lower current high switching frequency converter was chosen for the ±12V rails.



The input voltage requirement for the ±12V rails was set by the -12V rail and the output current requirement was about 150mA, calculated from the output voltage and 50Ω output impedance plus the current used by the op-amps which were comparatively minimal. According to [TI application note SNVA856B](https://www.ti.com/lit/pdf/snva856) for designing inverting buck-boost converters using buck ICs, the maximum voltage required by the buck regulator should be more than Vin + |Vout|. Thus the chosen buck converter IC must support Vin ≥ 24+13 = 37V. from these parameters, the [AP64060](https://www.diodes.com/datasheet/download/AP64060_AP64062.pdf) was selected.



The 3.3V rail supplies the MCU, the external DAC and the digital potentiometer.
According to the [STM32H723 datasheet](https://www.st.com/resource/en/datasheet/stm32h723zg.pdf) the maximum sum of current draw into all VDD pins of the microcontroller should be 620mA
and supply current in run mode at 550MHz with all peripherals enabled and cache off at Tj = 105°C is 430mA. With the cache off, the current consumption drops. Using the power consumption calculator in STM32CubeMX with the selected peripherals, the current consumption should be approximately 140mA. The external DAC [DAC60501](https://www.ti.com/lit/gpn/DAC60501) requires 2mA max. The digital potentiometer [MCP45HV51](https://ww1.microchip.com/downloads/aemDocuments/documents/MSLD/ProductDocuments/DataSheets/MCP45HVX1-Data-Sheet-DS20005304.pdf) requires a max of 650uA when the serial interface is active. Therefore an output current rating of 1A should be enough. Using these parameters the TI [LMR51610](https://www.ti.com/lit/gpn/LMR51610) was chosen.



The buck converter components were then selected according to the detailed design procedure section in the respective datasheets.

#### LDOs

The LDO reduces the noisy buck converter outputs into clean outputs. For the +3.3V rails, the [AP2112](https://www.diodes.com/datasheet/download/AP2112.pdf) was chosen for its low cost, high availability and 600mA output capability. For the +12V rails, the [TPS7E7201](https://www.ti.com/lit/gpn/TPS7E72) was chosen for its low cost and high PSRR. Th -12V LDO was chosen to be [SGM2209](https://www.sg-micro.com/rect/assets/84258ee4-63da-4fa3-ab9b-4afba55c79de/SGM2209.pdf) from the significantly smaller selection of negative LDOs because if its high adjustable output voltage and low noise.





## Analog signal conditioning

The analog signal conditioning circuit brings the 0V-2.5V DAC output, AC couples it with a 1Hz high pass filter, passes it through a 100Khz DAC reconstruction filter and amplifies it to turn it into a 20Vpp signal, which then gets attenuated before being buffered with a 50Ω output impedance and sent to the output BNC jack.

Initially thought of using a op amp voltage buffer, however realised that 20Vpp over a 50Ω termination resistor and 50Ω load results in 200mA of current output, which the TLV2888 cannot handle as the short circuit current is approximately 42mA. Therefore either a discrete transistor buffer or a buffer IC is required. Using a discrete buffer was chosen to maximise learning potential.

### Component selection

#### Op-Amps

The [TLV2888](https://www.ti.com/lit/gpn/TLV2888) was chosen for its wide operating voltage range, rail to rail operation, high precision, low cost and availability

#### Digital Potentiometer

The [MCP45HV51](https://ww1.microchip.com/downloads/aemDocuments/documents/MSLD/ProductDocuments/DataSheets/MCP45HVX1-Data-Sheet-DS20005304.pdf) was one of the only digital potentiometers with a wide operating voltage of more than 24V. A physical potentiometer was discounted because of the USB controllable requirement. A motorised potentiometer was deemed to be too expensive, low availability and bulky. A fixed step attenuator though a demultiplexer was also considered, but was also discounted because the demultiplexer required with wide voltage range was almost the same price or more expensive than the digital potentiometer.

#### Buffer

Class B push-pull and class AB with no biasing, with diode biasing, with resistor biasing and a diamond buffer was considered for the buffer stage. the class B buffer introduced too much distrotion around the crossover point. Diode biasing was simpler but still had some residual distortion. Resistor biasing requires experimentation with resistor values or a trim potentiometer to eliminate crossover distortion to minimise quiescent current and distortion due to the manufacturing tolerances. Thus the diamond buffer was chosen. I used [TI AN227](https://www.ti.com/lit/an/snoa725a/snoa725a.pdf) to understand the the structure of the diamond buffer. I then used KiCAD's built in ngspice simulator with the manufacturer's spice models for the BJTs to simulate the buffer.



## Microcontroller

The STM32H723 was chosen because it was the fastest microcontroller with an easily available development board.

