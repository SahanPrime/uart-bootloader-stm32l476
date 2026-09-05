# Memory Map

## Flash Layout

| Region      | Start Address | End Address  | Size  |
|-------------|---------------|--------------|-------|
| Bootloader  | 0x08000000    | 0x08007FFF   | 32 KB |
| Application | 0x08008000    | 0x080FFFFF   | 992 KB|
# Memory Map

## Flash Layout

| Region      | Start Address | End Address  | Size  |
|-------------|---------------|--------------|-------|
| Bootloader  | 0x08000000    | 0x08007FFF   | 32 KB |
| Application | 0x08008000    | 0x080FFFFF   | 992 KB|

Total flash on the STM32L476RG: 1024 KB (1 MB), starting at `0x08000000`
(the fixed flash base address for this chip, per RM0351).

## Why this split, and why 32K

The bootloader and application are two separate programs that must coexist
permanently in the same physical flash chip, so they need non-overlapping
address ranges — this is fundamentally different from RAM, where only one
program actually executes at a time (see `design-notes.md` for the
reasoning on why RAM is *not* split the same way).

**Why 32K for the bootloader specifically:**
- Must be a multiple of the flash page size (2 KB on the STM32L4), since
  flash erase operations work on whole pages — an unaligned boundary
  would either leave a gap or risk the erase routine touching bytes
  outside its intended region. 32 KB = 16 pages exactly.
- Large enough to comfortably hold the bootloader's actual code: current
  build size is ~4960 bytes (`.text` + `.data`) out of the 32K budget,
  leaving headroom for the UART receive protocol and flash-write logic
  still to be added in later weeks.
- Small enough to leave the vast majority of flash (992 KB) available
  for the application itself.

## SRAM

Both the bootloader and application projects use the **full 96 KB** of
SRAM1 (`0x20000000`–`0x20017FFF`), unsplit. Since only one program
executes at any given time (the bootloader runs to completion, then
hands off control to the application — they never run concurrently),
there's no need to partition RAM between them the way flash is
partitioned. Whichever program is currently executing owns all of RAM
for its own stack and variables.

`RAM2` (32 KB at `0x10000000`) exists as a separate physical SRAM
instance on this chip but isn't used by either project.

## Vector Table Relocation (VTOR)

When the bootloader jumps to the application, the CPU's `SCB->VTOR`
register must be updated to point at the application's vector table
(`0x08008000`) instead of the bootloader's (`0x08000000`) — otherwise,
any interrupt that fires after the jump would incorrectly look up
handler addresses from the bootloader's table.

**Decision:** VTOR relocation is handled in the **application's**
`system_stm32l4xx.c`, via `USER_VECT_TAB_ADDRESS` /
`VECT_TAB_OFFSET = 0x8000`, rather than explicitly in the bootloader's
jump function.

**Tradeoff accepted:** this introduces a small window — between the
bootloader's jump and the application's `SystemInit()` call — during
which `SCB->VTOR` still points at the bootloader's vector table, even
though the application's code is already executing. In practice this is
low-risk for this project, since interrupts aren't enabled early in the
application's startup sequence, but it's a real gap compared to setting
VTOR explicitly in the bootloader's jump function immediately before
transferring control (which would close the gap entirely, at the cost
of a bit more logic living in the bootloader rather than the
application).

## The Jump Sequence (`go2APP()`)

From the bootloader's perspective, jumping to the application is
functionally equivalent to a fresh reset — the application was compiled
assuming it gets its own clean stack pointer and its own vector table
active, the same setup the CPU normally provides automatically at
power-on. The bootloader has to manually replay that setup for the
application:

1. **Validity check** — read the value at the application's start
   address (`0x08008000`) and confirm it looks like a plausible SRAM
   address (masked check against `0x20000000`) before attempting to
   jump. Prevents jumping into blank/corrupted flash if no valid
   application has been programmed.
2. **Read the application's initial stack pointer** — the first word at
   `0x08008000`.
3. **Set the CPU's main stack pointer** to that value via `__set_MSP()`.
4. **Read the application's reset handler address** — the second word,
   at `0x08008004`.
5. **Jump** — cast that address to a function pointer and call it,
   transferring control into the application's `Reset_Handler`, which
   then performs its own `.data`/`.bss` initialization and calls the
   application's `main()`.
Total flash on the STM32L476RG: 1024 KB (1 MB), starting at `0x08000000`
(the fixed flash base address for this chip, per RM0351).



Both the bootloader and application projects use the **full 96 KB** of
SRAM1 (`0x20000000`–`0x20017FFF`), unsplit. Since only one program
executes at any given time (the bootloader runs to completion, then
hands off control to the application — they never run concurrently),
there's no need to partition RAM between them the way flash is
partitioned. Whichever program is currently executing owns all of RAM
for its own stack and variables.
