# BANE Firmware

This firmware aims to improve the usability of the Waldorf Protein Synthesizer, as well as add some features.

# Features

- Remapping of the ADSR envelope controls
- Added a Latch Function
- Flavor knob is now a compressor
- Mod matrix is now accessible by **pressing** the LFO "source" or "target" knobs


## Remapping of ADSR envelope controls
The envelope controls have been remapped to bypass a massive speed bump in the workflow when creating patches
- "Glide" knob is now "Attack" (Glide is now the "Shift" function)
- "Wave" knob is now "Decay"
- "Filter" knob is now "Sustain" (Pan is still the "Shift" function)
- "Amp" knob is now "Release" (Volume is still the "Shift" function)
- "Select" knob changes which envelope you are editing

## Added a "Latch" function
- Press the Effect "Type/Param" knob to enable the Latch function
  - Latch allows for up to 4 notes to be held simultaneously.
    - Press the latch knob repeatedly to choose the number of notes you would like to latch.
    - When the maximum number of notes +1 is pressed, the oldest note is released.
  - Latch works with the sequencer and arp
    - Sequencer will only latch the newest note played
    - Arpeggiator will latch up to 4 played notes
  - Turning Latch off will release the played notes

 ## Flavor Knob is now a Compressor
 The flavor knob has been repurposed as a 3rd effect: Compressor.
 - This compressor is a 3rd effect, separate from FX1 and FX2.
 - Audio Routing Chain: **Voices -> FX1 -> FX2 -> Master EQ -> Master volume -> Flavour compressor -> Stock limiter -> Hard clip at ±1.0 -> Output**
 - The compressor settings are not editable on the synthesizer, it has set values determined by the location of the "Flavor" knob:

| Flavour | 0% | 25% | 50% | 75% | 100% |
|---|---|---|---|---|---|
| Threshold (dB) | 0 | -12.5 | -25 | -37.5 | -50 |
| Ratio | 1:1 | 2.5:1 | 13.4:1 | 42.8:1 | 100:1 |
| Output gain (dB) | 0 | +4.6 | +18.9 | +31.4 | +43.5 |
| Attack (ms) | 4 | 1.6 | 0.63 | 0.25 | 0.1 |
| Release (ms) | 300 | 228 | 173 | 132 | 100 |

## Mod matrix is now accessible by **pressing** the LFO "source" or "target" knobs
The encoders for LFO "Source/Shape" and "Target" are stepped encoders. To access the mod matrix page, you need to turn the knob one full step. This can sometimes change the value.
- Pressing either encoder now accesses the mod matrix without having to turn the knob.
- These knobs did not previously have any "press" functionality, and no features were replaced or lost.

 

# Installation

1. Download the files
2. Plug your synth into the computer
3. Turn on your synth in bootloader mode (Hold "Shift" and turn on the synth)
4. Locate the "Protein" drive on your computer
5. Copy the PROTEIN.UF2 file to your computer, so you can go back to original firmware if you do not like the new one
6. Delete the "PROTEIN.UF2" file. [Note: The synth will rename any file to PROTEIN.UF2 after reboot. The BANE.UF2 file will be renamed to PROTEIN.UF2 next time you plug it into the computer.]
7. Paste the "BANE.UF2" file you downloaded
8. When the synth says "Ready", turn the synth off, then on again
9. The screen should say "Version: Bane"
10. Done!

# Usage

The synth works exactly the same as it did before, with the added features. Refer to Protein Feature Map.png for the layout
![Protein Feature Map](Protein%20Feature%20Map.png)

# Future Additions Planned
- Windows tool for adding custom wavetables
- "Wave" knob shift function is unused - add something else here


# Disclaimer

This is an unofficial, modified firmware. I did not create the original firmware or the Waldorf Protein, and this project is not affiliated with or endorsed by Waldorf.

I am not a programmer. The changes were made with the help of AI and have not been fully tested. The firmware may be unstable.

This firmware is provided "as is", without warranty of any kind. Use it at your own risk. Installing it may damage your synthesizer or void your warranty, and I am not responsible for any damage or loss.

This firmware is intended ONLY for the Waldorf Protein. Do not install it on any other device.

To return to the official firmware, download it from the Waldorf website and install it the usual way.
