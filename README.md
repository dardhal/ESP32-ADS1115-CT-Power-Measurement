# Summary : Multi channel AC power monitoring for home electrical circuits
# Description : By using SCT-013 CT clamps with 1 V AC output and external ADS1115 ADC, 
#               measure per-circuit AC current for instantaneous and cumulative energy utilization
# Software Used : Latest ESPHome code with two 4-channel ADS1115 ADC 16-bit boards attached to separate I2C buses
#   https://esphome.io/devices/esp32.html
#   https://esphome.io/components/i2c.html
#   https://esphome.io/components/sensor/adc.html
#   https://esphome.io/components/sensor/ct_clamp.html

Plan was to use a ESP32 with 6 built-in 12-bit ADCs to measure AC current through CT clamps for the internal circuits summarized below :
"""
--------------------------------------------------------------
TO MAXIMIZE AMOUNT OF CIRCUITS TO BE MEASURED WITH MINIMUM CHANGES
--------------------------------------------------------------
1. Frigorífico    :  5A clamp : 1150 W : Pending Purchase

2. Iluminación    : 15A clamp : 3000 W : Pending PIA Rellocation (Lavadora 1 to the right)
   Lavadora       :

3. Telecom.       :
   Sensores       :  5A clamp : 1150 W : Pending Purchase
   Ventilación    :

4. Fuerza Seco    : 10A clamp : 2300 W : Pending Installation

5. Fuerza Húmedo  : 30A clamp : 6900 W : Pending Installation
   Horno          :

6. Inducción      : 30A clamp : 6900 W : Pending Installation
   Lavavajillas   :
--------------------------------------------------------------
"""

On the ESP32 pins GPIO32 through GPIO39 can be used as ADCs, as described in the official vendor documentation :
https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc.html

Pins can measure up to 1.1 V which would fit perfectly the CT clamps (SCT-013) range of 0 - 1 V AC RMS (+-1.5 V peak), with 2.5dB attenuation.

However, we found the hard way (i.e. by trying to make it work and not achieving the expected results) the ESP32 ADC input pins are extremely non-linear, specially at low voltages, in brief, ADCs are unable to read any voltage below ~100-150 mW, and hence at least the lower 10% of the whole clamp measurement range can't be used.

For SCT-013 clamps with 1 V output, it means currents up to 15% of the full range can't be measured, or : 
- 0.75 A AC or  170 W for  5 A clamps
- 1.50 A AC or  345 W for 10 A clamps
- 3.00 A AC or  690 W for 20 A clamps
- 4.50 A AC or 1035 W for 30 A clamps

So even with the smallest of clamps we can't measure at all things such as standby lighting consumption, ventilation, etc. which makes for a significant part of the total energy consumption.

More details about the issue may be read here :
https://github.com/espressif/esp-idf/issues/164
https://www.esp32.com/viewtopic.php?t=2881

Even the official docs seem to admit this as a problem which is not resolved yet (linearily has been improved, but still ESP32 ADCs can't read anything below 100 mV) :
https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc.html#adc-attenuation



The alternative chosen was to use an external quality ADC (Texas Instruments ADS1116), which has 4 channels of 16 bit resolution and can be accessed through I2C. As the board may be set up with up to four different I2C addresses, the end result is up to four boards, or 16 individual ADC channels, may be configured on a single ESP32 board.
https://esphome.io/components/sensor/ads1115.html

NOTE : A ESP8266 only has one (software) I2C whereas a ESP32 has two hardware I2C only. So if using separate I2C buses for the ADC boards, the limit would be 8 ADC channels, however, by setting different addresses for each ADS1115 board, a single I2C should be good to address up to four ADS1115, simplifying cabling.
"""
#### ESP32 Hardware I2C bus
# There are two hardware I2C peripherals with identifiers 0 and 1. Any available output-capable pins can be used for SCL and SDA but the defaults are given below.
#     |  I2C(0)  |  I2C(1)
# ----+----------+--------
# scl |    18    |    25
# sda |    19    |    26
"""


#### When measured with the volt meter (in AC RMS mode), we obtained the following values at the terminals of the CT clamps for known currents (V AC) :
####
####              |  5 A CT Clamp  |  10A CT Clamp  |  15A CT Clamp  | 30A CT Clamp  |
#### -------------|----------------|----------------|----------------|---------------|
#### 0.135 A AC   |      0.0256 V  |      0.0130 V  |      0.0085 V  |     0.0042 V  |
#### 4.050 A AC   |      0.8090 V  |      0.4060 V  |      0.2700 V  |     0.1350 V  |
#### 7.700 A AC   |      1.5450 V  |      0.7750 V  |      0.5150 V  |     0.2580 V  |


