# Blueprints for handling switch button presses
Please see the blueprints above for Inovelli Blue and Red series switches. Blueprints support all Blue and Red Series switches, although I have not personally tested all of the Red models. Blue Series blueprint is only compatible with ZHA.

## Confirmed supported models
### Blue Series (ZHA)
* Dimmer (VZM31-SN)
* Fan (VZM35-SN)
* On/Off (VZM30-SN)
* mmWave (VZM32-SN)

### Red Series (Z-Wave)
* mmWave (VZW32-SN)
* Other models _should_ work, but have not been tested

## Special thanks
Blue Series blueprint is an adaptation of @fxlt’s fantastic [Inovelli VZM31-SN Blue Series 2-1 Switch blueprint](https://community.home-assistant.io/t/zha-inovelli-vzm31-sn-blue-series-2-1-switch/479148) to support more models than just the Dimmer switch. As with the original blueprint, it supports ZHA.

Red Series blueprint adapts the Blue Series blueprint for Z-Wave with help from [rohankapoorcom's blueprint](https://github.com/rohankapoorcom/homeassistant-config/blob/27ac149c16a2aadfb59e1fe790824dc181581348/blueprints/automation/rohankapoorcom/inovelli-vzw31-sn-red-series-switch.yaml).
