# UART Bootloader — STM32L476RG

A UART-based bootloader with firmware update capability, built on the
STM32L476RG (Nucleo-L476RG). Written to demonstrate strong C fundamentals:
manual flash programming, linker script / memory map control, vector table
relocation, and a custom UART packet protocol.

## Status
🚧 In progress — see `docs/design-notes.md` for current state.

## Motivation

Most deployed embedded products can't be reflashed with a hardware
programmer (ST-Link/JTAG) after they ship — they're sealed, remote, or in a
customer's hands. A bootloader turns firmware updates into a software
operation over an interface the device already has (here, UART), instead of
requiring physical debugger access.

## Memory Map

See [`docs/memory-map.md`](docs/memory-map.md).

## Protocol

See [`docs/protocol-spec.md`](docs/protocol-spec.md).

## Repo Layout

```
bootloader/     Bootloader firmware (runs first, handles updates, jumps to app)
application/    Example application firmware (runs at the offset flash address)
host-tool/      Python script to send firmware over UART
docs/           Memory map, protocol spec, design notes
```

## Building & Flashing

TBD — filled in once the toolchain setup is finalized.

## Design Decisions

See [`docs/design-notes.md`](docs/design-notes.md).
