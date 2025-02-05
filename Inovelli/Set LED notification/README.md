# Set Inovelli Blue/Red LED Notification (ZHA/Z-Wave) script
A Home Assistant script allowing you to define custom presets for the LED notifications on the Home Assistant Blue/Red Series switches using ZHA or Z-Wave.

## Pre-requisites 
* [Inovelli Blue or Red Series switches](https://inovelli.com/)
  * Note: This script has not been tested with all Red Series switches, so it may require further updates for some models. VZW32-SN is confirmed to work.
* Home Assistant
* A Zigbee (ZHA) or Z-Wave network

## How to use
1. Copy the code for the script ([Set Inovell Blue/Red LED Notification.yaml](set_inovelli_blue_red_led_notification_zha_zwave.yaml)) to a new script in Home Assistant.
2. You can use the default presets, but to customize your own:
   1. Edit the list of presets in the **Define presets** variables. All properties match the standard Inovelli Blue LED properties. Set led_number to -1 to target all LEDs at once.
   2. In the preset field, expand the selector options and edit the list to match the custom presets you just configured
3. You can now call this script in any of your automations!
