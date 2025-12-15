# LED indicators using single LED

This fork of the [poor man's LED indicator](https://github.com/BlueDrink9/zmk-poor-mans-led-indicator) allows it to be used together with caksoylar's rgb led widget in the same build without interfering with each other.

It is possible then to build for different boards together - I build for a n!n clone as well as a xiao ble together and the n!n clone uses this poor man's LED indicator to show when I'm persisting in a layer, and the Xiao can use the full fledged rgb led widget.

The code changes to this repository include the creation of a boards/shields subdirectory so that the module is not automatically imported, and within the leds.c all global symbols were prefixed with pmli_ to create a unique namespace for the poor-mans-led-indicator.

To build for zmk-rgbled-widget and zmk-poor-mans-led-indicator running together refer to my [personal keyboard repository](https://github.com/eyesonly/urchin-zmk-firmware). Note that the build.yaml looks like this:

  - board: xiao_ble
    shield: urchin_dongle rgbled_adapter
    snippet: studio-rpc-usb-uart
    cmake-args: -DCONFIG_ZMK_STUDIO=y
    artifact-name: xiao_dongle
  - board: nice_nano@2.0.0
    shield: urchin_dongle poor_mans_led
    snippet: studio-rpc-usb-uart
    cmake-args: -DCONFIG_ZMK_STUDIO=y
    artifact-name: urchin_dongle

Note how west.yaml imports the repository. And within boards/shields there is unique nice_nano.conf defined for the board.

The overlay logic for bringing in the user led setting only for this module is defined in boards/shields in this repository however. Please fork this repository and adjust if the overlay is to be used on a different board.

.......


This is a ZMK module containing a simple widget that utilizes a (typically built-in) LED controlled by a single GPIO.
It is used to indicate battery level, layer, and BLE connection status in a very minimalist way.

If you have access to RGB LEDs, addressed by 3 GPIOs (e.g. on Xiao BLE), you're probably better off using [caksoylar's rgb led widget](https://github.com/caksoylar/zmk-rgbled-widget/), which this is based off.

## Usage

To use, first add this module to your `config/west.yml` by adding a new entry to `remotes` and `projects`:

```yaml west.yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: bluedrink9
      url-base: https://github.com/bluedrink9
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: zmk-poor-mans-led-indicator
      remote: bluedrink9
      revision: main
  self:
    path: config
```

If you are building locally, see the instructions for [building with external modules](https://zmk.dev/docs/development/build-flash#building-with-external-modules)
in ZMK docs.

## Features

Currently the widget can do the following:

### Indicate battery on boot

If `CONFIG_INDICATOR_LED_SHOW_BATTERY_ON_BOOT=y`:

- Blink on boot depending on battery level (for both central/peripherals), thresholds set by `CONFIG_INDICATOR_LED_BATTERY_LEVEL_*` for HIGH, LOW and CRITICAL. Blink repeat counts set by `INDICATOR_LED_BATTERY_*_BLINK_REPEAT`, defaults in brackets below.
By default:
- High battery = blink (2) times slowly,
- low battery = blink (4) times at a medium pace,
- critical battery = blink (6) times frantically.

If `CONFIG_INDICATOR_LED_SHOW_CRITICAL_BATTERY_CHANGES=y`:

- Blink quickly once on every battery level change if below critical battery level (`CONFIG_INDICATOR_LED_BATTERY_LEVEL_CRITICAL`)

### Indicate BLE connection status changes

If `CONFIG_INDICATOR_LED_SHOW_BLE=y`, on every BT profile switch (on central side for splits):
- Blink medium pace n times for connected (where n is the profile number + 1),
- Blink slowly and constantly for open (advertising),
- Blink for a second one time if the profile is disconnected,

If `CONFIG_INDICATOR_LED_SHOW_PERIPHERAL_CONNECTED=y`:
- Blink twice quickly for connected, once slowly for disconnected on the peripheral side of splits

Most changes cannot be shown on peripheral, since events are not synced.
[This PR](https://github.com/zmkfirmware/zmk/pull/2036) implements message passing
between the halves and might one day be usable to fix this.

### Indicate layer changes

Enable `CONFIG_INDICATOR_LED_SHOW_LAYER_CHANGE` to show the highest active layer on every layer change
using a sequence of N frantic blinks, where N-1 is the zero-based index of the layer.

<!--Note that this can be noisy and distracting, especially if you use conditional layers.-->
<!--Configure `CONFIG_INDICATOR_LED_MIN_LAYER_TO_SHOW_CHANGE` to the-->
<!--zero-based index of the lowest layer you want this to apply to.-->
<!---->
Blink events are queued up to a maximum of 3 blink sequences, so one-shots and nested layers will show as
multiple sets of blinks.

You can also configure an array of layer values for which the LED
will stay lit at the end of its indication sequence. This is
helpful to know when you are still/stuck in a higher layer, when
you have set up layer toggle buttons.

## Configuration

See the [Kconfig file](Kconfig) for all of the available config properties, with descriptions. These will be more complete and up to date than the above readme.
You can add these settings to your keyboard conf file to modify the config values, e.g. in `config/hummingbird.conf`:

```ini
CONFIG_INDICATOR_LED_INTERVAL_MS=250
CONFIG_INDICATOR_LED_BATTERY_LEVEL_HIGH=50
CONFIG_INDICATOR_LED_BATTERY_LEVEL_CRITICAL=10
```

## Adding support in custom boards/shields

To be able to use this widget, you need at least one LED controlled by GPIOs (_not_ smart LEDs).
Once you have these LED definitions in your board/shield, simply set an `aliases` entry to `indicator-led`.

As an example, here is a definition for the user LED (connected to GND and separate GPIO) of a Nice!Nano and clones (e.g. Supermini nRF52840):

```dts
/ {

    leds {
        compatible = "gpio-leds";
        user_led: led_0 {
            gpios = <&gpio0 15 GPIO_ACTIVE_HIGH>;
            label = "User LED";
        };
    };

    aliases {
        indicator-led = &user_led;
    };
};
```

Finally, turn on the widget in the board's `.conf`:

```ini
CONFIG_INDICATOR_LED_WIDGET=y
```
