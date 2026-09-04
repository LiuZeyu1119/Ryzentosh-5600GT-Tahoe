# Ryzentosh 5600GT

Public-safe OpenCore guide for installing macOS Tahoe on a PC built around the AMD Ryzen 5 5600GT APU.

This guide describes the EFI structure and configuration choices used for this build. It intentionally does not publish serial numbers, MLB values, ROM values, UUIDs, Apple account data, or any other machine-identifying `PlatformInfo` values.

## IMPORTANT

This guide is complimentary to using [https://dortania.github.io/OpenCore-Install-Guide/](Dortania's) guide and [https://chefkiss.dev/guides/hackintosh/](ChefKiss') guide. I've used a mix of both to achieve this goal.

Take some time, read through the sections and keep following the guides, they will work!

The repository contains the basic structure of the EFI I used, but the files are only placeholders, except for the `config.plist`, which is almost complete, except for the sensitive information.

DON'T COPY THE EFI FOLDER AND TRY TO USE IT, **IT WILL NOT WORK**!

## Hardware Target

| Component | Target |
| --- | --- |
| CPU | AMD Ryzen 5 5600GT, 6 cores / 12 threads |
| iGPU | Radeon Graphics / Cezanne Vega iGPU, accelerated with `NootedRed.kext` |
| Ethernet | Realtek RTL8111/RTL8168 family, driven by `RealtekRTL8111.kext` |
| Bootloader | OpenCore |
| macOS | macOS Tahoe |

This EFI is board-specific. USB mapping, PCI paths, ACPI tables, and network device properties may need to be regenerated if your motherboard, BIOS, or port layout differs.

## Repository Contents

This private repository now contains two EFI sets:

```text
EFI/                    Original public-safe reference skeleton
EFI-5600GT-Tahoe/EFI/   Generated bootable EFI with real OpenCore binaries
```

The `.efi`, `.aml`, and `.kext` entries under the original `EFI/` directory are plain text placeholders. They exist to show the expected paths and to point to the correct download or generation source.

Use `EFI-5600GT-Tahoe/EFI/` for testing. It contains the actual OpenCore files, drivers, ACPI tables, kexts, and the default B450/X470/X570 configuration. Alternative motherboard profiles are under `EFI-5600GT-Tahoe/Profiles/`; read `EFI-5600GT-Tahoe/使用说明.md` before copying it to an EFI partition.

The generated configuration contains unique local SMBIOS identifiers. Keep this repository private and regenerate those identifiers before sharing or making the repository public.

## What Works

- macOS Tahoe boot through OpenCore
- AMD Ryzen CPU support through AMD kernel patches
- Radeon integrated graphics acceleration through `NootedRed.kext`
- Realtek Ethernet through `RealtekRTL8111.kext`
- USB mapping through `USBToolBox.kext` and `UTBMap.kext`
- CPU name display through `RestrictEvents.kext`
- OpenCore timeout booting to the selected macOS default

Apple services require valid, unique SMBIOS data and a built-in primary network interface. Do not reuse identifiers from another machine or from a public repository.

## EFI Layout

Expected OpenCore layout:

```text
EFI
|-- BOOT
|   `-- BOOTx64.efi
`-- OC
    |-- ACPI
    |-- Drivers
    |-- Kexts
    |-- Resources
    |-- Tools
    |-- config.plist
    `-- OpenCore.efi
```

### ACPI

```text
SSDT-EC.aml
SSDT-USB-Reset.aml
SSDT-USBX.aml
SSDT-XOSI.aml
```

### Drivers

```text
OpenRuntime.efi
HfsPlus.efi
ResetNvramEntry.efi
```

### Kexts

Recommended load order:

```text
Lilu.kext
VirtualSMC.kext
AppleMCEReporterDisabler.kext
NootedRed.kext
RealtekRTL8111.kext
SMCProcessorAMD.kext
SMCRadeonSensors.kext
RestrictEvents.kext
USBToolBox.kext
UTBMap.kext
```

Use release builds for daily use. Keep debug builds only for troubleshooting.

## BIOS Settings

Start with these settings, then adapt to your board:

- Disable Secure Boot.
- Disable CSM / legacy boot.
- Disable Fast Boot.
- Enable Above 4G Decoding.
- I have enabled Resizable BAR, even though it is supposed to work only with dGPUs.
- Enable XHCI handoff when the option exists.
- Use AHCI mode for SATA.
- Set integrated graphics UMA/framebuffer memory to at least 1 GB; 2 GB is preferred for the Ryzen APU. I have mine set to 2GB.
- Boot the installer in UEFI mode.

## Build the Installer

1. Create the macOS Tahoe USB installer with Apple's `createinstallmedia` command.
2. Create an EFI partition on the USB if it is not already present.
3. Copy the OpenCore `EFI` folder to the USB EFI partition.
4. Add the ACPI tables, drivers, and kexts listed above.
5. Generate `config.plist` from the matching OpenCore sample, then snapshot the ACPI, driver, tool, and kext entries with a plist-aware editor.
6. Add the AMD kernel patches from AMD-OSX/AMD_Vanilla and set the core-count patches for 6 physical cores.
7. Validate the config before booting:

```bash
plutil -lint /Volumes/EFI/EFI/OC/config.plist
```

Also validate with the `ocvalidate` binary from the same OpenCore release you are using.

## Required Config Notes

### AMD Kernel Patches

Ryzen systems need the AMD kernel patches. For the Ryzen 5 5600GT, set the physical core count to `06` in the AMD core-count patches. Use the patch set that supports macOS Tahoe and your OpenCore release.

Do not use the thread count. The Ryzen 5 5600GT has 6 physical cores, not 12 for this patch.

### Graphics

Use `NootedRed.kext` for the Ryzen 5 5600GT integrated Radeon graphics.

Do not load conflicting graphics fixup kexts such as `WhateverGreen.kext` or `NootRX.kext` with this setup.

### CPU Name

Use `RestrictEvents.kext` to show the CPU name correctly.

Add these NVRAM values under:

```text
NVRAM -> Add -> 4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102
```

```text
revpatch    String    cpuname
revcpu      Number    1
revcpuname  String    AMD Ryzen 5 5600GT
```

Set:

```text
PlatformInfo -> Generic -> ProcessorType = 1537
```

If editing XML directly, `revcpu` must be an integer tag:

```xml
<key>revcpu</key>
<integer>1</integer>
```

### Ethernet as Built-In

Apple services usually expect the primary network interface to be `en0` and built-in. For this board, the Realtek PCI path is:

```text
PciRoot(0x0)/Pci(0x2,0x1)/Pci(0x0,0x2)/Pci(0x7,0x0)/Pci(0x0,0x0)
```

Add this device property:

```text
DeviceProperties -> Add
  PciRoot(0x0)/Pci(0x2,0x1)/Pci(0x0,0x2)/Pci(0x7,0x0)/Pci(0x0,0x0)
    built-in    Data    01
```

In XML plist form:

```xml
<key>PciRoot(0x0)/Pci(0x2,0x1)/Pci(0x0,0x2)/Pci(0x7,0x0)/Pci(0x0,0x0)</key>
<dict>
  <key>built-in</key>
  <data>AQ==</data>
</dict>
```

After reboot, verify:

```bash
ioreg -rd1 -c IOEthernetInterface | grep -A18 -B5 '"BSD Name" = "en0"'
```

Expected result:

```text
IOBuiltin = Yes
```

### Boot Args

For daily use, keep `boot-args` empty unless you are actively debugging.

Useful temporary debug args:

```text
-v keepsyms=1 debug=0x100
```

Remove them once the system is stable.

### PlatformInfo and Privacy

Since we're not supposed to publish real `PlatformInfo` values, generate your own complete SMBIOS set locally. At minimum, keep these values private:

```text
SystemSerialNumber
MLB
SystemUUID
ROM
SystemMemoryStatus, if customized
ApECID, if used
PasswordHash
PasswordSalt
```

Before committing or sharing an EFI, inspect:

```text
PlatformInfo -> Generic
NVRAM -> Add
NVRAM -> Delete
Misc -> Security
```

Do not commit a config copied from a live system unless the identifiers have been replaced with safe placeholders.

## Installation Flow

1. Boot from the USB installer through OpenCore.
2. In Disk Utility, erase the target disk as APFS with GUID Partition Map.
3. Start the Tahoe installer.
4. Let the machine reboot through all installer phases. Always choose the macOS installer or target macOS volume from the OpenCore picker.
5. Complete Setup Assistant.
6. Mount the internal disk's EFI partition.
7. Copy the working USB `EFI` folder to the internal EFI partition.
8. Reboot from the internal drive.
9. Reset NVRAM once from the OpenCore picker.
10. Select the macOS entry and press `Ctrl + Enter` to save it as the default boot option.

## Post-Install Checks

Validate the config:

```bash
plutil -lint /Volumes/EFI/EFI/OC/config.plist
```

Check graphics acceleration:

```bash
system_profiler SPDisplaysDataType
```

Look for Metal support and normal VRAM instead of a low fallback value.

Check Ethernet:

```bash
ifconfig en0
ioreg -rd1 -c IOEthernetInterface | grep -A18 -B5 '"BSD Name" = "en0"'
```

Check loaded kexts:

```bash
kmutil showloaded | grep -E 'Lilu|VirtualSMC|NootedRed|RestrictEvents|RealtekRTL8111|USBToolBox'
```

## Troubleshooting

### OpenCore halts on a critical error

Validate `config.plist` with `plutil` and `ocvalidate`. Common causes are invalid XML tags, missing kext executables, bad paths, or config keys that do not match the OpenCore version.

### CPU name shows as Unknown

Confirm `RestrictEvents.kext` is enabled and that the `revpatch`, `revcpu`, and `revcpuname` NVRAM values are present.

### Apple Account says it cannot communicate with the server

Confirm the active network interface is `en0` and built-in. The expected `IOBuiltin` value is `Yes`.

Also confirm your SMBIOS identifiers are unique and valid. Do not use shared identifiers.

### Graphics acceleration is missing

Confirm `NootedRed.kext` is present, enabled, and loaded. Confirm UMA memory is at least 1 GB in BIOS. Remove conflicting kexts such as `WhateverGreen.kext` and `NootRX.kext`.

### USB ports do not behave correctly

Regenerate the USB map for your exact motherboard and case ports. `UTBMap.kext` is hardware-specific.

### A release EFI stops booting after kext updates

Keep OpenCore, `config.plist`, drivers, and kexts aligned. Update in small batches, validate the config, and keep a known-good USB EFI before rebooting.

## Maintenance

- Keep a bootable USB backup before changing the internal EFI.
- Update OpenCore and kexts together where possible.
- Replace whole `.kext` bundles instead of merging folder contents.
- Run `ocvalidate` after every OpenCore config change.
- Keep debug boot args out of the daily EFI.
- Keep PlatformInfo private in every public commit.

## References

- OpenCorePkg: <https://github.com/acidanthera/OpenCorePkg>
- AMD-OSX kernel patches: <https://github.com/AMD-OSX/AMD_Vanilla>
- NootedRed documentation: <https://chefkissinc.github.io/applehax/nootedred/>
- RestrictEvents: <https://github.com/acidanthera/RestrictEvents>
- VirtualSMC: <https://github.com/acidanthera/VirtualSMC>
- Lilu: <https://github.com/acidanthera/Lilu>
- USBToolBox: <https://github.com/USBToolBox/kext>
