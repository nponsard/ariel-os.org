+++
date = '2026-02-04T15:02:45+01:00'
draft = false
title = "Ariel OS v0.3.0: BLE, Sensors, UART, and More!"
# tags = ["Release"]
+++

Today we are delighted to announce a new version of Ariel OS!
With over 6 months of work and 500 merged PRs, this new version brings shiny new features and improvements.
These include upgraded Embassy and esp-hal dependencies, UART support, a brand new way to write and access sensor drivers, Bluetooth Low Energy support, structured board descriptions, and much more.

On top of those new features, this release adds support for more microcontrollers and boards, and some of those were added by the community!

You can check out the [full changelog of Ariel OS v0.3.0](https://github.com/ariel-os/ariel-os/blob/v0.3.0/CHANGELOG.md#030---2026-02-02).

Let's go over some of the highlights of this release.

## Upgraded Embassy and esp-hal

This was the major blocker for this new release, making everything work with as little breakage as possible: most applications built for Ariel OS v0.2.1 should work with Ariel OS v0.3.0 with little to no modifications.
You can check the full list of breaking changes in [the changelog](https://github.com/ariel-os/ariel-os/blob/v0.3.0/CHANGELOG.md#030---2026-02-02).

Ariel OS v0.3.0 publicly uses `esp-hal` v1.0.0, `embassy-nrf` v0.8.0, `embassy-rp` v0.8.0, `embassy-stm32` v0.4.0.

With esp-hal having stabilized as 1.0, upgrades to newer versions should come faster.
Embassy has not reached this milestone yet, but now has a faster release schedule that should make upgrades faster and easier for us.

## UART Support

Ariel OS now provides HAL-agnostic UART drivers on the four supported microcontroller families.
You can check if your board is supported [in the hardware support chapter of our book](https://ariel-os.github.io/ariel-os/dev/docs/book/hardware-functionality-support.html)
This allows you to easily send and receive data through UART, without having to worry about which hardware and HAL will be used.

Basic usage looks like this (full example in [`tests/uart-loopback`](https://github.com/ariel-os/ariel-os/tree/v0.3.0/tests/uart-loopback)):

```rust
#[ariel_os::task(autostart, peripherals)]
async fn main(peripherals: pins::Peripherals) {
    let mut config = hal::uart::Config::default();
    config.baudrate = Baudrate::_115200;
    let mut rx_buf = [0u8; 32];
    let mut tx_buf = [0u8; 32];
    let mut uart = pins::TestUart::new(
        peripherals.uart_rx,
        peripherals.uart_tx,
        &mut rx_buf,
        &mut tx_buf,
        config,
    )
    .expect("Invalid UART configuration");

    const OUT: &str = "Test Message";
    let mut input = [0u8; OUT.len()];
    uart.write_all(OUT.as_bytes()).await.unwrap();
    uart.flush().await.unwrap();

    uart.read_exact(&mut input)).await;
}
```

## The Sensor Abstraction

After a few rounds of design, the new sensor abstraction API is now available for you to try out!

All sensors using this system are can be accessed through a central registry and a universal API.
Want to know the current temperature? Just select any sensor in the registry that has the category `Temperature`.

```rust
#[ariel_os::task(autostart)]
async fn read_temperature() {
    let sensor = REGISTRY
        .sensors()
        .find(|s| s.categories().contains(&Category::Temperature))
        .unwrap();

    sensor.trigger_measurement().unwrap();
    let samples = sensor.wait_for_reading().await.unwrap();
    
    for (reading_channel, sample) in samples
        .samples()
        .filter(|(reading_channel, _)| reading_channel.label() == Label::Temperature)
    {
        info!(
            "Current temperature: {}{}",
            sample.value().unwrap(),
            reading_channel.unit()
        );
    }
}
```

You can find a more detailed example in [`examples/sensors-debug`](https://github.com/ariel-os/ariel-os/tree/v0.3.0/examples/sensors-debug) or [`examples/thermometer`](https://github.com/ariel-os/ariel-os/tree/v0.3.0/examples/thermometer) and read more about using and implementing sensor drivers in the [`sensors` module documentation](https://ariel-os.github.io/ariel-os/dev/docs/api/ariel_os/sensors/index.html).

Currently the sensor drivers have to be initialized manually in the application.
The long term goal is to have the sensor initialization code automatically integrated into the app and the list of sensors to be initialized would be handled in the board descriptions.

Speaking of board descriptions...

## Structured Board Description

Structured Board Description (SBD) is a new format we are introducing to describe how components on a board are wired to the MCU.
Currently supported components are LEDs, buttons, and USB ports.
This allows us to have some simple conditions to filter hardware: need an LED to display the status of your application? Select the `has_leds` module.
Need some basic interaction? Select the `has_buttons` laze module.
The access to LEDs and buttons is also generated: you can access them using `ariel_os_boards::pins::LedPeripherals` and `ariel_os_boards::pins::ButtonPeripherals`, using the macro `ariel_os::hal::group_peripherals!` when your task needs peripherals from multiple peripheral structs.

Our [blinky example](https://github.com/ariel-os/ariel-os/tree/v0.3.0/examples/blinky) is now even simpler, the `pins.rs` file with manual pins declaration is not needed anymore!

```rust
#[ariel_os::task(autostart, peripherals)]
async fn blinky(peripherals: ariel_os_boards::pins::LedPeripherals) {
    let mut led0 = Output::new(peripherals.led0, Level::Low);

    loop {
        led0.toggle();
        Timer::after_millis(500).await;
    }
}
```

## Bluetooth Low Energy

This release also adds supports for Bluetooth Low Energy (BLE).
On [supported hardware](https://ariel-os.github.io/ariel-os/dev/docs/book/hardware-functionality-support.html) (check for the "Bluetooth Low Energy" column) the Bluetooth hardware is automatically initialized when selecting a BLE-related laze module: `ble-peripheral` or `ble-central`.
You can then obtain a [`trouble_host::Stack`](https://docs.embassy.dev/trouble-host/0.5.0/default/struct.Stack.html) using [`ariel_os::ble::ble_stack().await`](https://ariel-os.github.io/ariel-os/dev/docs/api/ariel_os/ble/fn.ble_stack.html) and interact with this `Stack` to use the Bluetooth functionality.

Here's a simple task that advertises a "peripheral", you can find more detailed examples in [`examples/ble-advertiser`](https://github.com/ariel-os/ariel-os/tree/v0.3.0/examples/ble-advertiser) and [`examples/ble-scanner`](https://github.com/ariel-os/ariel-os/tree/v0.3.0/examples/ble-scanner):

```rust
#[ariel_os::task(autostart)]
async fn run_advertisement() {
    let mut host = ariel_os::ble::ble_stack().await.build();
    let mut adv_data = [0; 31];
    let len = AdStructure::encode_slice(
        &[
            AdStructure::CompleteLocalName(b"Ariel OS BLE"),
            AdStructure::Flags(LE_GENERAL_DISCOVERABLE | BR_EDR_NOT_SUPPORTED),
        ],
        &mut adv_data[..],
    )
    .unwrap();
    
    let _ = join(host.runner.run(), async {
        let params = AdvertisementParameters::default();
        let _advertiser = host
            .peripheral
            .advertise(
                &params,
                Advertisement::NonconnectableScannableUndirected {
                    adv_data: adv_data.get(..len).unwrap(),
                    scan_data: &[],
                },
            )
            .await;
        loop {
            info!("Advertising");
            Timer::after_secs(60).await;
        }
    })
    .await;
}
```

You can check the [dedicated page of the book](https://ariel-os.github.io/ariel-os/dev/docs/book/bluetooth.html) to learn more about BLE in Ariel OS.

## Hardware Support

Hardware support is growing with addition from the core team and the community!

- [@royb3](https://github.com/royb3) added support for the STM32H753ZI MCU and the ST NUCLEO-H753ZI board.
- [@bugadani](https://github.com/bugadani) added support for the ESP32-S2, ESP32-S2Fx2, ESP32-S2Fx4, ESP32-S2Fx4R2 MCUs, the ESP32-S2-SOLO-2 hardware module, and the Espressif ESP32-S2-DevKitC-1 and ESP32-C3-DevKit-RUST-1 boards.
- [@baptleduc](https://github.com/baptleduc) added support for the Seeed Studio LoRa-E5 mini board.
- [@dgtlrift](https://github.com/dgtlrift) added support for the Heltec WiFi LoRa 32 V3 board.
- [@therealprof](https://github.com/therealprof) added support for the STM32F042K6 MCU and the ST NUCLEO-F042K6, BBC micro:bit V1 boards.
- [@Ollrogge](https://github.com/Ollrogge) added support for the FPU on compatible Cortex-M MCUs.
- [@SimonIT](https://github.com/SimonIT) added support for `getrandom` on MCUs that have a HWRNG.

Thank you for those contributions, we also want to thank [@KiyoLelou10](https://github.com/KiyoLelou10) and [@tshepang](https://github.com/tshepang) for their contributions to other parts of Ariel OS.

## Getting Started With Ariel OS

Want to try this new version for yourself?
Head over to our [Getting started guide](https://ariel-os.github.io/ariel-os/dev/docs/book/getting-started.html) and start your Ariel OS journey today!
