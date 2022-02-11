# Animating icons on Nextion Display
Instructions for showing animating icons on the Sonoff NSPanel's Nextion display.  

To spice up the visuals on my customised NSPanel, I wanted to have the washing machine icon reflect the operational status of the device.  
My goal was to:
1) animate whenever the machine is washing
1) eye-catching visual when it has finished (clothing needs to be removed)
1) and when it's off.

> Disclaimer, the NextionEditor has a built-in animation studio, which could probably be used to achieve something similar as I describe below. However, this is how I did it.

## Home Assistant
I monitor the washing machine's power consumption in Home Assistant using a Blitzwolf SHP-2 (`entity_id:esp02`), flashed with ESPHome. 

It turns out my washing machine draws a little over 1.6 Watt when idle and obviously more when actually running. This allows me to attach some states to the device based on power draw.
```yml
# template sensor
- platform: template
  sensors:
    washing_machine_status:
      friendly_name: 'Washing Machine Status'
      value_template:  >
        {% if states.sensor.esp02_power.state | float > 2.0 %}
          {{ "Running" }}
        {% elif states.sensor.esp02_power.state | float > 1 %}
          {{ "Idle" }}
        {% else %}
          {{ "Off" }}
        {% endif %}

```

## Creating images

Create a set of images that make up each frame the animation. I took the washing machine image from the Material Design Icon set and rotated the door part with 45° for each frame. 

## Nextion Editor

### Importing images
After importing the various images that make up the animation and machine state, I ended up with the following list
|ID|Image|File|
|-|-|-|
|![washing-machine-on-1.png](https://github.com/dulfer/sonoff-nspanel-esphome/blob/df0d42b67b13580e245adc9a98a3a7e162deeb9a/nextion-hmi/icons/washing-machine-on-1.png?raw=true)|19|`Machine Running 1`|
|![washing-machine-on-2.png](https://github.com/dulfer/sonoff-nspanel-esphome/blob/df0d42b67b13580e245adc9a98a3a7e162deeb9a/nextion-hmi/icons/washing-machine-on-2.png?raw=true)|20|`Machine Running 2`|
|![washing-machine-on-3.png](https://github.com/dulfer/sonoff-nspanel-esphome/blob/df0d42b67b13580e245adc9a98a3a7e162deeb9a/nextion-hmi/icons/washing-machine-on-3.png?raw=true)|21|`Machine Running 3`|
|![washing-machine-on-4.png](https://github.com/dulfer/sonoff-nspanel-esphome/blob/df0d42b67b13580e245adc9a98a3a7e162deeb9a/nextion-hmi/icons/washing-machine-on-4.png?raw=true)|22|`Machine Running 4`|
|![washing-machine-idle.png](https://github.com/dulfer/sonoff-nspanel-esphome/raw/main/nextion-hmi/icons/washing-machine-idle.png?raw=true)|23|`Machine Idle`|
|![washing-machine-off.png](https://github.com/dulfer/sonoff-nspanel-esphome/blob/df0d42b67b13580e245adc9a98a3a7e162deeb9a/nextion-hmi/icons/washing-machine-off.png?raw=true)|24|`Machine Off`|

### Page objects
|Toolbox item|Id||
|-|-|-|
|Picture|`pWash`||
|Timer|`tmWashing`|Execute script at intervals|
|Variable|`vaWashing`|Keep track of active image id|

### Timer Event code
The timer is set to a 500ms interval.  
A variable `vaWashing` is there to keep track of the active image id, initial value set to 0.

As I have to cycle through 4 images, I just increment the value of the variable `vaWashing`, then check if it's more than 3 and set it to 0 to start at the first image again. Increment that value with 19 to get an image id from 19 to 22.
```c
pWash.pic=vaWashing.val+19
vaWashing.val++
if(vaWashing.val>3)
{
  vaWashing.val=0
}
```


## ESPHome
With the Home Assistant sensor in place, it is easy to act on any change in state of the sensor.  
Whenever the state is `Running`, start the timer which start the animation. On either `Idle` or the *else-state* stop the timer and set icon to id of idle (23) or off (24) image.
```yml
text_sensor:
  # ....
  - platform: homeassistant
    id: info_button_wasmachine
    entity_id: sensor.washing_machine_status
    on_value:
      then:
        - lambda: |- 
            if (id(info_button_wasmachine).state == "Running") {
              id(disp1).send_command_printf("tmWashing.en=1");
            } else if (id(info_button_wasmachine).state == "Idle" )  {
              id(disp1).send_command_printf("tmWashing.en=0");
              id(disp1).send_command_printf("pWash.pic=23");
            } else {
              id(disp1).send_command_printf("tmWashing.en=0");
              id(disp1).send_command_printf("pWash.pic=24");
            }
        - component.update: disp1         
```