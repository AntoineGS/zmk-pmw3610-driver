PMW3610 driver implementation for ZMK with at least Zephyr 3.5

This work is based on
[ufan's implementation](https://github.com/ufan/zmk/tree/support-trackpad) of
the driver.

> [!WARNING]
> This driver has only been tested with a ProMicro nrf52840 + PMW3610 trackball
> board.

## Improvements made

- !! Reduced battery comsumption in deep sleep by 5.5x on the trackball board !!
- Support for ZMK 0.3
- Addition of mouse acceleration, two modes available: sigmoid and quadratic

## Installation

Only GitHub actions builds are covered here. Local builds are different for each
user, therefore it's not possible to cover all cases.

Include this project on your ZMK's west manifest in `config/west.yml`:

```yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: antoinegs
      url-base: https://github.com/AntoineGS
  projects:
    - name: zmk
      remote: zmkfirmware
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: antoinegs
      revision: main
  self:
    path: config
```

Then, edit your `build.yml` to look like this, 3.5 is now on main:

```yml
on: [workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

Now, update your `board.overlay` adding the necessary bits (update the pins for
your board accordingly):

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 1, 13)>,
                <NRF_PSEL(SPIM_MOSI, 0, 10)>,
                <NRF_PSEL(SPIM_MISO, 0, 10)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 1, 13)>,
                <NRF_PSEL(SPIM_MOSI, 0, 10)>,
                <NRF_PSEL(SPIM_MISO, 0, 10)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 9 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 11 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

        /*   optional features   */
        // snipe-layers = <1>;
        // scroll-layers = <2 3>;
        // automouse-layer = <4>;

        /*   optional: ball action on specific layers  */
        // arrows {
        //     layers = <3>;
        //     bindings =
        //         <&kp RIGHT_ARROW>,
        //         <&kp LEFT_ARROW>,
        //         <&kp UP_ARROW>,
        //         <&kp DOWN_ARROW>;
        //
        //     /*   optional: ball action configuration  */
        //     tick = <10>;
        //     // wait-ms = <5>;
        //     // tap-ms = <5>;
        // };
    };
};

/ {
  trackball_listener {
    compatible = "zmk,input-listener";
    device = <&trackball>;
  };
};
```

Now enable the driver config in your `board.config` file (read the Kconfig file
to find out all possible options):

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_MOUSE=y
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000
CONFIG_PMW3610=y
CONFIG_PM_DEVICE=y
CONFIG_PMW3610_PM=y
CONFIG_PMW3610_ACCELERATION_ALGORITHM=2 # 0: disabled, 1: quadratic, 2: sigmoid
CONFIG_PMW3610_ACCELERATION_SENSITIVITY=10 # 1-100
```
