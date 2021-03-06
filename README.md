![Azure DevOps builds (branch)](https://img.shields.io/azure-devops/build/albertogeniola/c4128d1b-c23c-418d-95c5-2de061954ee5/1/0.3.X.X.svg)
![Deployment](https://albertogeniola.vsrm.visualstudio.com/_apis/public/Release/badge/c4128d1b-c23c-418d-95c5-2de061954ee5/1/1)
![Test status](https://img.shields.io/azure-devops/tests/albertogeniola/Meross/1?failed_label=failed&label=Tests%20&passed_label=passed&skipped_label=skipped)
[![PyPI version](https://badge.fury.io/py/meross-iot.svg)](https://badge.fury.io/py/meross-iot)
[![Downloads](https://pepy.tech/badge/meross-iot)](https://pepy.tech/project/meross-iot)
![PyPI - Downloads](https://img.shields.io/pypi/dm/meross-iot.svg?label=Pypi%20Downloads)
[![Beerpay](https://beerpay.io/albertogeniola/MerossIot/badge.svg?style=flat)](https://beerpay.io/albertogeniola/MerossIot)

# Meross IoT library
A pure-python based library providing API for controlling Meross IoT devices over the internet.

To see what devices are currently supported, checkout the *Currently supported devices* section. 
However, some devices _might work as expected even if they are not listed_ among the supported devices. 
In such cases, you're invited to open an issue and report tbe working/non-working status of your device.
This will help us to keep track of new devices and current support status of the library.


This library is still work in progress, therefore use it with caution.

## Installation
Due to the popularity of the library, I've decided to list it publicly on the Pipy index.
So, the installation is as simple as typing the following command:

```
pip install meross_iot==0.3.1.9 --upgrade
```

## Usage
The following script demonstrates how to use this library.

```python
from meross_iot.manager import MerossManager
from meross_iot.meross_event import MerossEventType
from meross_iot.cloud.devices.light_bulbs import GenericBulb
from meross_iot.cloud.devices.power_plugs import GenericPlug
from meross_iot.cloud.devices.door_openers import GenericGarageDoorOpener
from random import randint
import time
import os


EMAIL = os.environ.get('MEROSS_EMAIL') or "YOUR_MEROSS_CLOUD_EMAIL"
PASSWORD = os.environ.get('MEROSS_PASSWORD') or "YOUR_MEROSS_CLOUD_PASSWORD"


def event_handler(eventobj):
    if eventobj.event_type == MerossEventType.DEVICE_ONLINE_STATUS:
        print("Device online status changed: %s went %s" % (eventobj.device.name, eventobj.status))
        pass

    elif eventobj.event_type == MerossEventType.DEVICE_SWITCH_STATUS:
        print("Switch state changed: Device %s (channel %d) went %s" % (eventobj.device.name, eventobj.channel_id,
                                                                        eventobj.switch_state))
    elif eventobj.event_type == MerossEventType.CLIENT_CONNECTION:
        print("MQTT connection state changed: client went %s" % eventobj.status)

        # TODO: Give example of reconnection?

    elif eventobj.event_type == MerossEventType.GARAGE_DOOR_STATUS:
        print("Garage door is now %s" % eventobj.door_state)

    else:
        print("Unknown event!")


if __name__ == '__main__':
    # Initiates the Meross Cloud Manager. This is in charge of handling the communication with the remote endpoint
    manager = MerossManager(meross_email=EMAIL, meross_password=PASSWORD)

    # Register event handlers for the manager...
    manager.register_event_handler(event_handler)

    # Starts the manager
    manager.start()

    # You can retrieve the device you are looking for in various ways:
    # By kind
    bulbs = manager.get_devices_by_kind(GenericBulb)
    plugs = manager.get_devices_by_kind(GenericPlug)
    door_openers = manager.get_devices_by_kind(GenericGarageDoorOpener)
    all_devices = manager.get_supported_devices()

    # Print some basic specs about the discovered devices
    print("All the bulbs I found:")
    for b in bulbs:
        print(b)

    print("All the plugs I found:")
    for p in plugs:
        print(p)
        
    print("All the garage openers I found:")
    for g in door_openers:
        print(g)
        
    print("All the supported devices I found:")
    for d in all_devices:
        print(d)
        
    # You can also retrieve devices by the UUID/name
    # a_device = manager.get_device_by_name("My Plug")
    # a_device = manager.get_device_by_uuid("My Plug")
    # Or you can retrieve all the device by the HW type
    # all_mss310 = manager.get_devices_by_type("mss310")
    # ------------------------------
    # Let's play the garage openers.
    # ------------------------------
    for g in door_openers:
        if not g.online:
            print("The garage controller %s seems to be offline. Cannot play with that..." % g.name)
            continue
        print("Opening door %s..." % g.name)
        g.open_door()
        print("Closing door %s..." % g.name)
        g.close_door()
        
    # ---------------------
    # Let's play with bulbs
    # ---------------------
    for b in bulbs:  # type: GenericBulb
        if not b.online:
            print("The bulb %s seems to be offline. Cannot play with that..." % b.name)
            continue
        print("Let's play with bulb %s" % b.name)
        if not b.supports_light_control():
            print("Too bad bulb %s does not support light control %s" % b.name)
        else:
            # Is this an rgb bulb?
            if b.is_rgb():
                # Let's make it red!
                b.set_light_color(rgb=(255, 0, 0))
            
            time.sleep(1)
            
            if b.is_light_temperature():
                b.set_light_color(temperature=10)
            
            time.sleep(1)
                                       
            if b.supports_luminance(): 
                # Let's dimm its brightness
                random_luminance=randint(10, 100)
                b.set_light_color(luminance=random_luminance)

        b.turn_on()
        time.sleep(1)
        b.turn_off()
        
    # ---------------------------
    # Let's play with smart plugs
    # ---------------------------
    for p in plugs:  # type: GenericPlug
        if not p.online:
            print("The plug %s seems to be offline. Cannot play with that..." % p.name)
            continue

        print("Let's play with smart plug %s" % p.name)

        channels = len(p.get_channels())
        print("The plug %s supports %d channels." % (p.name, channels))
        for i in range(0, channels):
            print("Turning on channel %d of %s" % (i, p.name))
            p.turn_on_channel(i)

            time.sleep(1)

            print("Turning off channel %d of %s" % (i, p.name))
            p.turn_off_channel(i)

        usb_channel = p.get_usb_channel_index()
        if usb_channel is not None:
            print("Awesome! This device also supports USB power.")
            p.enable_usb()
            time.sleep(1)
            p.disable_usb()

        if p.supports_electricity_reading():
            print("Awesome! This device also supports power consumption reading.")
            print("Current consumption is: %s" % str(p.get_electricity()))

    # At this point, we are all done playing with the library, so we gracefully disconnect and clean resources.
    print("We are done playing. Cleaning resources...")
    manager.stop()

    print("Bye bye!")
```

## Currently supported devices
Starting from v0.2.0.0, this library should support the majority of Meross devices on the market.
The list of tested devices is the following:
- MSL120 (RGB Bulb)
- MSS110 (Smart plug)
- MSS210 (Smart plug)
- MSS310 (Smart plug with power consumption)
- MSS310h (Smart plug with power consumption)
- MSS425e (Smart strip)
- MSS530H (Wall-mount switches)
- MSG100 (Garage opener)
- MSH300 (Smart hub + valve thermostat)

I'd like to thank all the people who contributed to the early stage of library development,
who stimulated me to continue the development and making this library support more devices:

Thanks to [DanoneKiD](https://github.com/DanoneKiD), [virtualdj](https://github.com/virtualdj), [ictes](https://github.com/ictes), [soberstadt](https://github.com/soberstadt), [ping-localhost](https://github.com/ping-localhost).

## Protocol details
This library was implemented by reverse-engineering the network communications between the plug and the meross network.
Anyone can do the same by simply installing a Man-In-The-Middle proxy and routing the ssl traffic of an Android emulator through the sniffer.

If you want to understand how the Meross protocol works, [have a look at the Wiki](https://github.com/albertogeniola/MerossIot/wiki). Be aware: this is still work in progress, so some pages of the wiki might still be blank/under construction.

## Homeassistant integration
Yeah, it happened. As soon as I started developing this library, I've discovered the HomeAssistant world.
Thus, I've decided to spend some time to develop a full featured Homeassistant custom component, that you find [here](https://github.com/albertogeniola/meross-homeassistant).
Thanks to @troykelly who made a wish and supported my efforts in developing such component! 

## Running on OSX?
If so, please make sure you did install client certificates. [Follow instructions here](https://github.com/albertogeniola/MerossIot/issues/62#issuecomment-535769621).

## Donate!
I like reverse engineering and protocol inspection, I think it keeps your mind trained and healthy. 
However, if you liked or appreciated by work, why don't you buy me a beer? 
It would really motivate me to continue working on this repository to improve documentation, code and extend the supported meross devices.

Moreover, donations will make me raise money to spend on other Meross devices. 
So far, I've bought the following devices:
- MSL120
- MSS210
- MSS310
- MSS425E
- MSS530H
- MSG100
- MSH300

By issuing a donation, you will:
1. Give me the opportunity to buy new devices and support them in this library
1. Pay part of electricity bill used to keep running the plugs 24/7 
(Note that they are used for Unit-Testing on the continuous integration engine when someone pushes a PR... I love DEVOPing!)  
1. You'll increase the quality of my coding sessions with free-beer!

[![Buy me a beer](http://4.bp.blogspot.com/-1Md6-deTZ84/VA_lzcxMx1I/AAAAAAAACl8/wP_4rXBXwyI/s1600/PayPal-Donation-Button.png)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=CQQCK3RN32BHL&source=url)

[![Beerpay](https://beerpay.io/albertogeniola/MerossIot/badge.svg?style=beer-square)](https://beerpay.io/albertogeniola/MerossIot)  [![Beerpay](https://beerpay.io/albertogeniola/MerossIot/make-wish.svg?style=flat-square)](https://beerpay.io/albertogeniola/MerossIot?focus=wish)

### Look at these babies!

<p>
Look at the test environment that ensures high quality code of the library!
</p>
<img src="ext-res/plugs/test-env.jpg" alt="Current test environemnt" width="400" />
<p>When a pull-request is performed against this repository, a CI pipeline takes care of building the code, 
testing it on Python 3.5/3.6/3.7, relying on some junit tests and, if all the tests pass as expected, the library
is released on Pypi. However, to ensure that the code <i>really works</i>,
the pipeline will issue on/off commands against real devices, that are dedicated 24/7 to the tests. 
Such devices have been bought by myself (with contributions received by donators). 
However, keeping such devices connected 24/7 has a cost, which I sustain happily due to the success of the library.
Anyways, feel free to contribute via donations! 
</p>

## Changelog
### 0.3.1.9
- Fixed missing method implementation
- Improved logging
- Minor improvements __str__ methods
### 0.3.1.8
- Added HUB + thermostat support
### 0.3.1.6
- Improved light bulb driver (capacity)
### 0.3.1.5
- Fixed wrong MerossEventType being advertised for DeviceDoorStatusEvent
### 0.3.1.3
- Added event fire capability to GenericBulb class.
- Fixed bulb state kwargs bug
- Improved set_light_color function for bulbs
### 0.3.0.2
- Fixed door closing checks when using the async + callback close() and open() methods.
### 0.3.0.1
- Added get_power_consumption() and get_electricity() methods as abstract methods of AbstractMerossDevice
- Fixed regression passing manager parameter when firing Meross events.
### 0.3.0.0rc4
- Added added switch_state to the generated event
### 0.3.0.0rc3
- Added quick fix for MSS560 color control
### 0.3.0.0rc2
- Fixed Major bugs with MSG100
- Updated README examples
### 0.3.0.0rc1
- Added MSG100 support
- Fixed errors being logged when power consumptionX command was issued on powerplugs
### 0.3.0.0b1
- General refactor of the library
- Added event-based support
- Fixed default mqtt broker address for non-european devices
### 0.2.2.1
- Added basic bulb support: turning on/off and light control
- Implemented MSL120 support
- Implemented MSL120 automatic test
- Extended example script usage to show how to control the light bulbs
- Added maximum retry limit for execute_command and connect()
### 0.2.1.1
- Code refactoring to support heterogeneous devices (bulbs, plugs, garage openers)
### 0.2.1.0
- Implemented auto-reconnect on lost connection
- Improving locking system in order to prevent library hangs when no ack is received
