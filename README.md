# VoodooHDA Bootloader Injector


## What the Injection Script does — explained

Since macOS Big Sur (11), injecting VoodooHDA.kext directly from the bootloader fails.
The booter (OpenCore AND Clover — same injection engine) cannot link VoodooHDA because
its parent classes live in Apple's IOAudioFamily.kext, which is NOT in the boot kernel
collection (BootKernelExtensions.kc). Result:

    OC: Prelinked injection VoodooHDA.kext () - Invalid Parameter

The script solves this automatically. Here is exactly what it does, step by step.

---

## The idea in one sentence

    Ship a RENAMED copy of Apple's IOAudioFamily.kext alongside VoodooHDA,
    and make VoodooHDA link against the renamed copy — the booter then
    resolves everything at prelink time.

---

## Step-by-step

### 1. Safety guards (the script exits early if anything is wrong)

- VoodooHDA.kext installed in /Library/Extensions ?
  -> EXIT. Bootloader injection and an installed kext cannot coexist.
     The script prints the removal + rebuild commands for you.

- VoodooHDA.kext already patched (previous run) ?
  -> DELETED, exit. Prevents double-patching a distributed kext.

- The script uses the kext provided in the ORIG-2.6.2 folder (unpatched VoodooHDA.kext) 
  -> Systematically regenerates the functional kext from this base.

### 2. Get a REAL IOAudioFamily.kext

Apple increasingly ships IOAudioFamily as a hollow stub (no binary inside —
verified on Ventura 13 and Tahoe 26). The script finds a real one in this order:

    a) /System/Library/Extensions  — only if the binary is really there
    b) Kernel Debug Kit — auto-downloaded from Dortania KdkSupportPkg:
         - matches your EXACT running build (sw_vers)
         - no exact match? asks you: closest build (risky) or abort
         - mounts the .dmg in /private/tmp
         - expands the .pkg inside (yes, the dmg contains a .pkg)
         - extracts IOAudioFamily.kext from the payload
         - deletes the dmg + temp files automatically

Only KDK (or real /S/L/E) binaries are in the required PRE-LINK state.
Extracting from SystemKernelExtensions.kc does NOT work — those binaries are
pre-relocated by Apple and hang the boot.

### 3. Patch the copy — the ALIAS trick

The copy is modified so it can coexist with Apple's stock IOAudioFamily:

    - binary thinned to x86_64 (arm64e slice removed)
    - identifier renamed EVERYWHERE:
        com.apple.iokit.IOAudioFamily  ->  net.voodoo.IOAudioFamily
      (inside the Mach-O binary AND in Info.plist)

Without the rename, the duplicate identifier stalls early boot (DriverKit stage).
With it, the kernel loads both stacks happily — tested and verified.

### 4. Patch VoodooHDA.kext

Its OSBundleLibraries is rewritten:

    com.apple.iokit.IOAudioFamily  ->  net.voodoo.IOAudioFamily

so at prelink time the booter resolves VoodooHDA's 191 audio imports
against the ALIASED copy injected just before it.

### 5. Sign both kexts (ad-hoc)

Both kexts are re-signed after patching, so signatures verify cleanly.

---

## What YOU do after the script

    1. Copy BOTH kexts to EFI/OC/Kexts (or EFI/CLOVER/kexts/Other)
    2. OC config: Kernel -> Add
         IOAudioFamily.kext   ABOVE   VoodooHDA.kext
       (array order = link order — wrong order = Invalid Parameter)
    3. No Kernel -> Block entries
    4. No other steps. No SIP changes — works with SIP fully enabled
   (verified: macOS Tahoe 26.7.1, OpenCore 1.0.7, csr-active-config = 00000000). 
    5. Reboot 

Verify:

    kmutil showloaded | grep -i voodoo
    system_profiler SPAudioDataType

---

## Why the ORDER matters (Kernel -> Add)

The booter resolves each injected kext's imports against the kexts injected
BEFORE it in the Kernel->Add array. IOAudioFamily first = VoodooHDA's classes
resolve. VoodooHDA first = nothing to resolve against = Invalid Parameter.

## Why the ALIAS matters

Two kexts with the same identifier cannot coexist. The stock IOAudioFamily
(lives in the SystemKC) plus our injected copy = collision at early userspace.
Renaming ours to net.voodoo.IOAudioFamily makes the kernel treat them as two
different kexts — AppleHDA keeps its family, VoodooHDA gets its own.

## Why KDK binaries matter

A kext binary extracted from a kernel collection has its relocations already
applied by Apple's kcgen. The booter needs PRE-LINK binaries (relocations
intact) to place them at a new address. KDK ships pristine pre-link binaries
— that is why the script refuses to use anything else.

---

## Tested

    VoodooHDA V-2.9.2  -> audio CONFIRMED (analog + HDMI), macOS 26.3
    VoodooHDA V-3.6.7  -> boots and loads (codec support varies per machine)
    OpenCore and Clover (shared OcAppleKernelLib engine)

## Credits

    chris1111  — alias technique, script, VoodooHDA maintainer
    Slice, AutumnRain, Zenith432  — original VoodooHDA developer
    Dortania   — KDK mirrors
    Acidanthera — OcAppleKernelLib (the injection engine we patched through)


