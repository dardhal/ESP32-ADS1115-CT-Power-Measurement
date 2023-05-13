# ESP32 + Current Transformer (CT) clamps power measuring using ESPHome

This is the way I have used [ESPHome](https://esphome.io/) with rather inexpensive hardware and some soldering iron to monitor instantaneous power consumption (and hence, cumulative energy usage) for 8 of my home electrical circuits (plus some others indirectly), which I feed to [Home Assistant](https://www.home-assistant.io/) (and then to [InfluxDB](https://www.influxdata.com/) and [Grafana](https://grafana.com/) for long-term storage and charting).

If following the description here, you may end up achieving some working implementation resembling better, proven and much more reliable solutions like [Circuit Setup Expandable 6-channel Energy Meter](https://circuitsetup.us/product/expandable-6-channel-esp32-energy-meter/?v=04c19fa1e772), at a fraction of the cost, by spending lots of time and patience.

Please do not try this at home if you are not familiar, comfortable or qualified to work with AC voltage. You may get yourself hurt or cause electrical damage.


# Software and hardware needed

As for software, you just need a working [ESPHome](https://esphome.io/) install. Get the latest version updated to and check for any errors or recommendations to upgrade from [PlatformIO](https://platformio.org/).

The choice for hardware consists of :

 - An appropriate development board. Even if the code may work with some adjustments on a simple ESP8266, it may lack the horsepower for the quick calculations needed, so save yourself some pain and got straight to any ESP32 development board. This will set you less than 10 € off.
 - One or more Texas Instruments ADS1115 4-channel 16 bit Analog to Digital Converter (ADC) boards. You need one channel for each electrical circuit you want to measure current and hence power for. You may find them as development boards with ping to be soldered for as low as 2 € each.
 - One Current Transformer (CT) clamp with 0 - 1 V output for each one of the circuits you want to measure current and power for. I chose SCT-013, which I purchased from [Aliexpress](https://com.aliexpress.com/). A few €uros each.

# How this works
CT clamps allow you to measure current going through a wire (Live or Neutral, but not both, as current going in both directions would cancel out the induction produced by current flowing through the wires) without having to cut the wire. Just open the clamp, put it around either the live or neutral wire, and close the clamp. For a CT clamp with a 10 A / 1 V rating, when no current is flowing, clamp with put 0 V on the output wires, and when current is 10 A AC, output will be 1 V (AC RMS), with intermediate currents resulting in proportional voltage.

This voltage may be read by an ADC and turned into a digital measurement, which can be used by any computer (or an ESP32) to easily calculate active power consumption (note this does not measure reactive power or DC power in any way). Once the ESP32 has calculated the power measured, it will report to Home Assistant through the common API, so you can use the values as sensors in Home Assistant.

# Hardware setup

Not getting into many details here. It is dead simple and you should already know how to do this before trying to go for the instructions in this repository, else you are playing a dangerous game here, which may end up with you or your home fried.

# ESPHome sketch

The whole-house-power-monitoring.yaml file in this repository includes abundant comments and should be self-explanatory for the most part. For some of the testing and considerations made before finally getting things to work, please read on.

# Some findings, errors and things we tried first before we arrived to the final implementation

## Lack of ESP32 ADC pins linearity
Initial plan was to use a ESP32 with its 6 built-in 12-bit ADCs channels / pins to measure AC current through CT clamps (with 0 - 1 V output voltage).

On the ESP32 development board I have, pins GPIO32 through GPIO39 can be used as ADCs, as described in the official vendor documentation :
https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc.html

Pins can measure up to 1.1 V which would fit perfectly the CT clamps (SCT-013) range of 0 - 1 V AC RMS (+-1.5 V peak), with 2.5dB attenuation set in ESPHome.

However, we found the hard way (i.e. by trying to make it work and not achieving the expected results) the ESP32 ADC input pins are extremely non-linear, specially at low voltages. In brief, ADCs are unable to read any voltage below ~100-150 mV, and hence at least the lower 10% of the whole clamp measurement range can't be used for anything close to reliable.

For SCT-013 Current Transformer (CT) clamps with 1 V output, those first 150 mV we can't reliably read off the ESP32 ADC pins means that, depending on the CT ampage, we can't read power below the following values, making the ESP32 ADC pins unsuitable for the purpose :
- 0.75 A AC, or  170 W for  5 A CT clamps
- 1.50 A AC, or  345 W for 10 A CT clamps
- 3.00 A AC, or  690 W for 20 A CT clamps
- 4.50 A AC, or 1035 W for 30 A CT clamps

So even with the smallest of clamps we can't measure at all things such as standby lighting consumption, ventilation, fridge, etc. which in my home make for a significant part of the total energy consumption.

More details about the issue may be read in the links below :
https://github.com/espressif/esp-idf/issues/164
https://www.esp32.com/viewtopic.php?t=2881

Even the official docs seem to admit this as a problem which is not resolved yet (linearity has been improved, but still ESP32 ADCs can't read anything below 100 mV) :
https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc.html#adc-attenuation

That's why I shifted to using external ADCs, namely, the Texas Instruments ADC1115, as mentioned above, which comes with an I2C, configurable addresses and better resolution (16-bit vs 12-bit for the ESP32 ADCs).

## ADS1115 and development board connection details
I would discourage using a ESP8266, although it may work (although I doubt it for memory and CPU being too small). And it comes with just one software I2C, whereas the ESP32 has two hardware I2C buses. Should I have used separate I2C buses for the two ADCs, I could have done so. But I chose to have a single bus to simplify wiring and "configure" the second ADC to use the secondary I2C address (ADS1115 boards are configured by default with 0x48 address, but can switch to using 0x49 (and some, additional two addresses) by connecting two pins together on the board.

There are two hardware I2C peripherals with identifiers 0 and 1. Any available output-capable pins can be used for SCL and SDA but the defaults are given below.

         |  I2C(0)  |  I2C(1)
     ----+----------+--------
     scl |    18    |    25
     sda |    19    |    26

## Could not make ADS1115 work in "differential mode" to measure current in both directions (to calculate power consumption and surplus from PV panels)
After all, I am not sure if it is really possible to measure power in both directions reliably using just an SCT-013 and an ADS1115 in differential mode :
- SCT-013 induces an AC voltage on its outputs , which is exactly the same no matter the current direction
- Differential ADS1115 does not seem to play a role or help at all in determining current direction (as it only measures voltage +- Volts)
- In any case, at any instant it should still return the absolute (positive) value (as calculated RMS) for either the current pulled from the grid or the current sent to the grid

Not a problem for me anymore, as I had a watmeter installed when upgrading my PV system to use battery storage, and can reliably get that information from the wattmeter now.

## The way current and power is calculated in the .yaml file is *NOT* as described in the ESPHome documentation
And that's because after trying it "the correct way" and investigating, I ended up doing things in a different way which, for me, worked fine and much more reliably than the way described in the several ESPHome related docs.

For me, using "continuous_mode" for the ADS1115 and all other configuration options for ESPHome ct_clamp current sensor, didn't yield the expected results. Very slow updating and unreliable (and mostly wrong) measurements across the scale. Even if initial testing with a voltmeter of a 10 A / 1 V CT clamp using a separate commercial power meter was promising :

    ct_clamp OUT //  SCT-013 output (mV)
    --------------------------------------------
    0.026 A AC   //   30 mV -> 0.300 A AC (load)
    0.060 A AC   //   65 mV -> 0.650 A AC (load)
    0.098 A AC   //  100 mV -> 1.000 A AC (load)
    0.710 A AC   //  485 mV -> 4.850 A AC (load)
    1.150 A AC   //  791 mV -> 7.910 A AC (load)

What worked for me in the end is what's described in this post in the Home Assistant forums :
https://community.home-assistant.io/t/zmpt101b-precision-voltage-sensor-module/124617/7

 * Connect the SCT-013 voltage output to ADS1115 ADC channel. Disable "continuous_mode" for ADS1115 ADCs in ESPHome
* Calculate AC voltage (RMS == Root Mean Square), using ESPHome "filter", instead of resorting to the "ct_clamp" fpr doing so
* Map the calculated AC voltage RMS (in the range of 0.0 - 1.0 ) to current depending on the specific SCT-013 (ie for a 20 A / 1 V sensor, 100 mV would map to 2 A in the measured circuit)
* Then template to calculate instantaneous power and integrate over time (daily) would be the same as if using "ct_clamp"

When using this approach I got faster response times to changes in AC load, more precise and more reliable measurements than using ADS1115 "continuous_mode" and ESPHome "ct_clamp", which is supposedly the way to go. Your mileage may vary.

YAML code for one of the circuits ended up looking like this :
 
 
```yaml
    # 4. POWER DRY (ADS1115 0x48, A3_GND)
    # 10 A / 1 V SCT-013 #1
    # Sockets in dry rooms, including home office and extractor hood
    # Note "ha_ac_voltage" is grid voltage as pulled from Home Assistant sensor for the same name
      - platform: ads1115
        multiplexer: 'A3_GND'
        gain: 2.048
        name: "Power Dry AC Current"
        ads1115_id: ads1115_bus_0x48
        update_interval: 24ms
        id: ac_a_powerdry_ct_sensor
        state_class: measurement
        device_class: current
        unit_of_measurement: "A"
        icon: "mdi:flash"
        accuracy_decimals: 8
        filters:
          - lambda: return x * x;                 ####
          - sliding_window_moving_average:        #
              window_size: 1250                   # This blocks calculates the RMS voltage sampled by the ADS1115 ADC (averages over 30 s and reports every 15 s)
              send_every:  625                    #
          - lambda: return sqrt(x);               ####
          - multiply: 10.0                        # Mapping measured voltage from CT clamp to current in the primary circuit (for 10 A / 1 V CT clamp)
          - offset: -0.1                          # Substracting 0.1 A AC calculated from result, as it is "electrical noise" induced by other clamps
    
      - platform: template
        name: "Power Dry AC Power"
        lambda: |-
          if ( id(ac_a_powerdry_ct_sensor).state < 0 ) {
            return 0;
          }
          if ( id(ha_ac_voltage).state < 180 ) {
            return ( abs(230.0 * id(ac_a_powerdry_ct_sensor).state ) );
          }
          else {
            return ( abs(id(ha_ac_voltage).state * id(ac_a_powerdry_ct_sensor).state ) );
          }
        accuracy_decimals: 1
        update_interval: 30s
        state_class: measurement
        device_class: power
        unit_of_measurement: "W"
        id: ac_power_powerdry
      
      - platform: total_daily_energy
        name: "Power Dry Energy Consumed"
        power_id: ac_power_powerdry
        unit_of_measurement: kWh
        filters:
          - multiply: 0.001
```

Regarding ADS1115 ADC and continuous_mode, this link may be worth checking and may explain my own experience :
https://cdn-learn.adafruit.com/downloads/pdf/adafruit-4-channel-adc-breakouts.pdf

> In continuous mode, the board will read the latest value that the
> ADS1x15 device has converted. This is faster as it does not have to
> wait for the ADC to finish converting the value, but it only really
> works when reading data from one of the four pins on the ADC.



# TODO

 - Could not figure out a way to separate lengthy and mostly repetitive configuration blocks to separate files (!include), which makes the .yaml file quite hard to read (sorry)


# Useful links

 - ESPHome ESP32 Platform : https://esphome.io/components/esp32.html
 - ESPHome I2C Bus Component : https://esphome.io/components/i2c.html
 - ESPHome ADS1115 4-Channel 16-Bit A/D Converter Sensor : https://esphome.io/components/sensor/ads1115.html?highlight=ads1115
 - ESPHome CT Clamp Current Sensor : https://esphome.io/components/sensor/ct_clamp.html


