# Notes - Waldorf Protein firmware project

**Maintained:** 2026-11-06. **Original analysis:** stock Waldorf Protein v1.02 through **Round 105**, static disassembly unless an older round explicitly says otherwise. **Current custom firmware:** see `Protein_Handoff.md` for accepted runtime features, source/hash provenance and tested versus untested claims.

**Reading order:** 1. `Protein_Handoff.md` for current status; 2. the newest numbered rounds near the end of this document for corrected stock-firmware findings; 3. earlier notes for method and historical development. Earlier findings sometimes contradict later corrections; use the most recent evidence. `Functions_Map.csv` contains the complete Round 105 stock application mapping (762 class P strong, 1,638 class L, 2 DATA). The map does **not** cover appended custom firmware functions.

**Patch record, 2026-11-06:** accepted experimental firmware is `PROTEIN_ADSR_SELECT_SHIFTWAVE_FIXED_EXPERIMENTAL.UF2` (reported SHA-256 `71d51b8a77f71dfa8b8a4da950be3de6ea20f70c9bcfb011e1da4bace4fc4849`). It uses Glide/Wave/Filter/Amp for selected envelope A/D/S/R; Select only cycles WAVE/FILTER/AMP while an ADSR screen is open; Shift+Wave is a reserved no-op; Shift+Glide retains factory glide; the Effects Type encoder press toggles a four-note FIFO latch, with `LATCH ON`/`LATCH OFF` notices. The graph has a fixed 44-pixel left shift and native values, not time units. There is no ADSR OFF option. User reports the build appears to work well in a clean emulator, **not physically tested**. Earlier reported QEMU caveat: entering Arp could clear current sounding latched notes while preserving latch state; not exhaustively retested. The technical patch history and validation limits are in `Protein_Handoff.md` sections 3-7.

**Current speculative idea, not a feature:** repurpose Flavour to control compressor behavior while retaining its popup; no design or firmware change approved. Original Flavour consumer analysis appears in Round 15 and the later Round 33 search.

---

## Original decompilation notes, preserved in chronological order

# Waldorf Protein v1.02 - reverse-engineering workspace

Machine-decompiled, not Waldorf source. Function names are ROM symbols, auto-names from log strings
(unverified), or the few hand-checked names listed below. Everything else is `fcn.<address>`.

## The firmware
- `PROTEIN.UF2` = UF2 container (family 0x12e6002f) holding one ESP32-S3 app image, 594,688 bytes.
- Project "Chutney_Firmware", v1.02_b2-232-gdda56c3c, built 21 Sep 2026, ESP-IDF v5.5.1, 8 MB flash.
- Not encrypted, no secure-boot signature block. Image ends with the standard checksum + SHA-256.
- The UF2 only carries the app. Bootloader, partition table and the "data"/NVS partitions (saved
  sounds, globals) are not in it.

## Memory layout (app image segments)
| Segment | Address | Size | Content |
|---|---|---|---|
| DROM | 0x3c060020 | 143,120 | strings, tables, wave ROM |
| DRAM | 0x3fc9da00 | 22,100 | initialised RAM data |
| IRAM | 0x40378000 | 31,364 | fast code (vectors, ISRs) |
| IROM | 0x42000020 | 340,856 | main code |
| IRAM | 0x4037fa84 | 56,964 | fast code |
| RTC | 0x50000000 | 32 | RTC data |

## Files
- `protein_v102.elf` - the image as an ELF with ROM + peripheral symbols. Open in Ghidra, IDA or r2.
  Byte-identical segments to the firmware.
- `decomp/ALL.c` - all 2,611 functions decompiled (2,576 OK, 35 failed). String and parameter
  addresses are annotated inline as comments.
- `PARAMS.md` / `analysis/params.json` - the full parameter table (108 entries).
- `analysis/functions.json` - every function: address, size, strings used, calls.
- `analysis/names*.r2` - names applied (r2 commands; easy to convert for Ghidra).
- `scripts/` - pipeline: `esp2elf.py`, `seed_funcs.py`, `pass1_analyze.py`, `params.py`, `pass2_decompile.py`.
- `setup_tools.sh` + `tools_patches/` - rebuild the decompiler, including a fix: stock Ghidra's
  Xtensa spec is missing the LX7 SALT/SALTU instructions the ESP32-S3 uses.

## Map so far
### Hardware input
- `Board__scanInputs` 0x42026c48 - polls 6 encoders, 15 pots, 16 buttons, 16 LEDs; logs under tag "Board".

### Parameters
- Descriptor table 0x3c071f54, 32 bytes/entry: name, label, steps, enum-label table, default, flags, formatter.
- `ParamMgr__getDescriptor` 0x42014f7c - ID to descriptor. Sound params: ID = 0x95 + part*0x53 + n.
  Envelope IDs (part 0): WaveEnv A/D/S/R 0xa8-0xab, AmpEnv A/D/S/R 0xb0-0xb3, FiltEnv A/D/S/R 0xb5-0xb8, GlideRate 0xa5.
- `ParamMgr__loadDefaults` 0x4201584c.

### Front panel knob sections (target for the ADSR mod)
- `FrontPanel__ctor_WaveFilterAmpMaster` 0x4201e108 builds the "Wave", "Filter", "Amp", "Master" (and "Pan")
  section objects. Each has its own vtable (Wave 0x3c073310, Filter 0x3c073340, Amp 0x3c073370,
  0x3c0733a0, ...). Slot 2 is the event handler, e.g. `AmpSection__onEvent` 0x4202006c
  (event-type meaning not confirmed yet). The 16-bit pairs stored there (0x13/0x14, 0x25/0x26, 0x20/0x21...)
  are probably the pot/control IDs each section owns - unconfirmed.

### Wavetables (target for custom wavetables)
- Wave ROM 0x3c079740: 300 single-cycle waves x 64 bytes, 8-bit (Microwave 1 style).
- Wavetable definitions 0x3c07e240: 61 x uint16 wave numbers per table (0xffff = interpolated slot).
- `MW1WavetableCache__buildTable` 0x4202b6c4 expands a table into the RAM cache.
- The cache already has RAM slots for 24 extra tables (numbers 0x20-0x37) and 122 extra waves
  (numbers 0xf6-0x16f). What fills them is not traced yet - this is the most promising route for
  custom wavetables.

### MIDI CC handling (session 2, decompile reading only, unconfirmed on hardware)
- `fcn.4201486c` 0x4201486c is a MIDI CC handler. Decompiler shows args as in_a3 = CC number, in_a4 = value (register naming is unreliable).
  - Stores every CC value as float (value/127) in a table at obj+0x42c (4 bytes per CC).
  - CC 0x40 (sustain pedal) calls object vtable slot 0x2c with (value >= 64). CC 0x7b (all notes off) calls slot 0x50. CC 0x78 (all sound off) calls slot 0x54.
  - CC 1, 2, 4, 11 get smoothed-value updates. CC 0x4a stores per-note-number data at obj+0xaac.
- `UserInterface__processMidiInput` 0x4201c830 only queues the message (FreeRTOS queue send); the parser is elsewhere.
- Other MIDI status-byte dispatch candidates to read next: ALL.c lines 54494, 56809, 59056, 64783.
- Global params live in a float array in the ParamMgr object (LayerMode at +0x3328; guess: array starts at +0x32f0, 4 bytes each). Not verified.
- Arp engine: no strings or named functions found. No latch parameter exists in the global table (idx 0-3 are Mode, Algo, Octaves, Reset). Not located yet.
- Wavetable user slots: not traced yet.

### Engine event path (session 2, decompile reading only, unconfirmed on hardware)
- `fcn.42019bdc` 0x42019bdc is a MIDI message router. It splits on status byte (0x80 off, 0x90 on, 0xa0 poly AT, 0xb0 CC -> `fcn.420199d0`, 0xc0, 0xd0, 0xe0) and builds a 24-byte event (type word first).
  - Guessed event types from the code: 0 = note on, 1 = note off, 5 = poly aftertouch, 4 = channel pressure. Types 2 and 3 are sent around note-on in a legato/steal branch (obj+0xc5c). Unconfirmed.
- `Engine__sendEngineEvent` 0x4200d52c (name auto-generated): takes the event on the stack, pre-processes some types (6 = param ID + value, 10, 11, 12, 13), then pushes it to a FreeRTOS queue (24-byte items) and logs under tag "Engine" if the push fails. Engine object is at 0x3fca3974.
- Only 6 functions call it, all MIDI/sysex/log related. The consumer (engine task that pops the queue) is not found yet; the arp is most likely behind that consumer.
- Next: find the xQueueReceive (0x40384e50 or 0x40385160 are heavily used FreeRTOS candidates, unverified) call that reads this queue.

### Wavetable cache trace (session 2, decompile reading only, unconfirmed on hardware)
- `MW1WavetableCache__buildTable` 0x4202b6c4 reads (cache object = arg1):
  - table numbers 0x20-0x37 (24 tables): definitions (61 x uint16) from RAM at obj+0x5627a, 0x7a bytes each.
  - wave numbers 0xf6-0x16f (122 waves): 64-byte waves from RAM at obj+0x56e02.
  - everything else: ROM wave data 0x3c079740 and ROM table definitions 0x3c07e240.
  - its output goes to obj+0x4 (0xf40 bytes per table) and a valid-flag array at obj+0x54d44 (0x3d bytes per table).
- Only one reader of those RAM regions was found (the builder). No code writing to obj+0x5627a or obj+0x56e02 by literal offset was found, so v1.02 may never fill them (leftover from the Microwave 1 design), or fills them through a computed pointer I did not find. Unconfirmed.
- `fcn.4201682c` 0x4201682c is a waveform draw/scan routine over the cache, not a writer.
- The Wtb1Table / Wtb2Table parameters have 64 steps. How a step maps to a table number is not traced, so even filled RAM tables may not be selectable without patching that mapping.
- Where the cache is stored across power cycles (NVS/data partition) is not in the UF2, so a user-wavetable save format is unknown.

### Wavetable numbering (session 2, decompile reading only, unconfirmed on hardware)
- 89 table slots, numbers 0x00-0x58. Names are an 88-entry pointer array in DRAM at 0x3fca05ac. Entries 32-55 are literally "User   1" to "User  24" (table numbers 0x20-0x37). Entries 56-87 are ROM tables SawSync1 ... Chorus 2.
- The Wavetable parameter (Wtb1Table idx 29, Wtb2Table idx 36) has 64 steps. Its display formatter `fcn.420177dc` 0x420177dc maps step s to table number s if s < 32, else s + 0x18. So steps 32-63 jump over the 24 User tables to 56-87. The User tables exist in the cache but cannot be selected from the knob.
- Table number ranges in the init loop `fcn.4202c500` 0x4202c500: 0x20-0x37 only interpolated (never built from ROM); 0x1c-0x4b except the user range and 0x58 are synthesized by `fcn.4202bb70` 0x4202bb70 (2447 bytes); the rest come from ROM via buildTable. `fcn.4202b5ac` fills 0xffff (empty) wave slots by averaging neighbours.
- The engine's own step-to-table mapping (audio side) is not located; the formatter is display only. If the engine uses the same mapping, exposing User tables means changing steps 64 -> 88 in both descriptors and removing the +0x18 shift in both places.
- Still needed for custom wavetables: code to write user table definitions (0x7a bytes each at obj+0x5627a) and waves (64 bytes each at obj+0x56e02), call buildTable for the table number, and a save/load path.

### Engine classes and the arp/seq/chord layer (session 2, decompile + radare2 disassembly, unconfirmed on hardware)
Four C++-style classes (vtables in DROM) make up the engine core:
| vtable | role (inferred) | key methods |
|---|---|---|
| 0x3c070110 | Note processor: arp + sequencer + chord layer, wraps a voice manager (child pointer at obj+8) | +0x14 noteOn `fcn.42014528` (340 B), +0x18 noteOff `fcn.42013e90` (174 B), +0x40 setParam `fcn.42013f40` (1148 B) |
| 0x3c0701a8 | Per-part voice manager (8 voices x 0x70 bytes at obj+0x10) | +0x14 noteOn `fcn.4037a4b0`, +0x18 noteOff `fcn.40380afc`, +0x1c controller `fcn.4201486c`, +0x2c sustain pedal `fcn.4204e728` |
| 0x3c070218 | Parameter manager | +0x0c getDescriptor, +0x10 loadDefaults |
| 0x3c070288 | Product/audio engine | +0x24 `fcn.4037bfb0` (10,092 B, probably the audio render) |
- Engine task `fcn.4200ccc8` pops 24-byte events and calls `fcn.4200d7f4` (decompile failed; read from disassembly). Event type -> engine vtable slot: 2 -> +0x28, 4 -> +0x1c (controllers incl. pressure), 6 (param ID, value) -> +0x40 setParam, 8 -> +0x48 (value x 0.01), 9 -> +0x4c. Note on/off types 0/1 not resolved (disassembly there desyncs).
- Hooks +0x58 and +0x60 in the voice manager vtable are empty stubs, and the engine task skips calling them.
- Sustain pedal is real at voice level: pedal flag at voice manager obj+0x398 turns note-off into a held state (flag value 2).
- Arp/seq parameters, as handled in `fcn.42013f40` (note processor object):
  - ArpSeqMode (id 0) -> obj+0x1b54 (previous mode at obj+0x1b58, 7000 decimal). Change from Off stops sounding notes via vtable +0x50; going to a non-Off mode calls `fcn.4201763c` (start); leaving calls `fcn.42017534` (stop). Flag obj+0x1b5c = arp/seq running; while set, note events are NOT forwarded to the voice manager.
  - ArpAlgo (1) -> +0x758, ArpOctaves (2) -> +0x760 (value+1), ArpReset (3) -> +0x748 and flag +0x74c.
  - RhythmPattern (4) -> +0xad4, BPM (5) -> float at +0x10, Gate (6) -> +0xac4, StepLen (7) -> +0xac8, Swing (8) -> +0xacc, SeqLen (9) -> +0xa09, ChordMode (0x10) -> +0x1be9.
- Held-key table: noteOn `fcn.42014528` sets obj+0xa0+note = 1 and stores velocity as float at obj+0x120+note*4 and timestamps at obj+0x320+note*8; noteOff `fcn.42013e90` sets obj+0xa0+note = 0. The arp most likely reads this table to build its pattern.
- Latch idea (not tested): in noteOff, skip clearing obj+0xa0+note when a latch flag is set and arp is running, and clear the table on the first new note-on after all keys were released. Needs the arp tick/readers found, a free byte for the latch flag, and somewhere to switch it (a button, or a CC).
- Caveat: the decompiler dropped the implicit `this` argument in several short functions (`fcn.4201763c`, `fcn.42017534`), so field names above come from the ones it did show.
- Tools used (not in the zip): radare2 6.2.4 built locally; a small capstone 6 + pyelftools script was unreliable on some functions. Use radare2 for disassembly.

### Front panel envelope knob groups (session 2, decompile reading only, unconfirmed on hardware)
- `AmpSection__onEvent` 0x4202006c only reacts to event types 2 and 3 and opens display pages ("Vel Amnt"). It is not the knob-to-parameter routing.
- The Wave, Filter and Amp panel sections each contain a "knob group" object (vtables 0x3c07325c, 0x3c07328c, 0x3c0732bc; shared handler `fcn.4201f0f0`, draw routine `fcn.4201f204` 0x4201f204).
- A group stores four 16-bit parameter indices at obj+0x24, +0x26, +0x28, +0x2a (Attack, Decay, Sustain, Release). It reads each value from the parameter manager at (part * 0x54 + index) and draws the "Att / Dec / Sus / Rel" envelope view. A flag at obj+0x2f switches display modes.
- Indices set in `FrontPanel__ctor_WaveFilterAmpMaster` 0x4201e108: Wave group 0x13-0x16, a group at +0x194/+0x198 with 0x20-0x23, a group at +0x11c/+0x120 with 0x25-0x28. If index = descriptor idx - 25, then 0x13-0x16 is the Wave envelope and 0x20-0x23 is the Filter envelope. The Amp envelope would be 0x1b-0x1e; 0x1b appears once at +0xf4. The 0x25-0x28 group does not match an envelope and is unresolved. The ctor decompile may be misattributing offsets between groups.
- Not yet found: how the physical pots (Board__scanInputs polls 15 pots) turn into parameter changes, and where a pot index is mapped to one of these groups. This is the piece needed to make the Glide/Wave/Filter/Amp knobs drive a selectable envelope.
- Idea (not tested): the A/D/S/R indices live in RAM objects, so a patch could rewrite those four 16-bit values at runtime to point a group at a different envelope.

### Knob input path (session 2, decompile + radare2, unconfirmed on hardware)
- Board object at 0x3fcd1b78. `Board__scanInputs` 0x42026c48 reads 6 encoders (obj+0x20, 0x18 bytes each), 15 pots (obj+0xb0, 4 bytes each; ADC value converted to 0-511 with a 12-count hysteresis in `fcn.42027278`), 16 buttons and 16 LEDs. Pot calibration state is at 0x3fcd1c28 (u16 value + u16 raw, 4 bytes per pot). `fcn.4202656c` is the calibration screen ("Turn all min+max").
- Five FreeRTOS queues at 0x3fccff54 (24-byte engine events), 0x3fccff58, 0x3fccff5c, 0x3fccff60, 0x3fccff64, created in `fcn.4201c868`. Their send wrappers (`fcn.4201c508`, `fcn.4201c4d8`, `fcn.4201c538`, `fcn.4201c7f8`) are called indirectly, so the call graph does not show the producers.
- Pump `fcn.4201c6c0` 0x4201c6c0 drains the queues: 24-byte events go to `Engine__sendEngineEvent`; other items go to a UART write (`fcn.42027fbc`, uart 1) and `fcn.42028260`.
- Panel event handler `fcn.4201e640` 0x4201e640 gets (control id, value, event type). Type 0 = button, 2 = pot, 3 = encoder (inferred). `fcn.4201e390` (type 2) only selects which section page to show: control id 1 -> +0x10, 2 and 7 -> Wave section +0x70, 5 -> +0xa0, 6 -> +0xf8 or +0x14c, 9 -> +0x170 or +0x128, 10-11 -> +0x1a0, 12 -> +0x1d8, others -> Filter section +0xd0. Every section then gets the same event for redraw.
- Still not found: the code that turns a pot value into a parameter change (event type 6 with parameter ID and float value, which the engine applies at vtable +0x40). It is probably behind one of the indirect queue writers.

### Pot to parameter writer FOUND (session 2, disassembly + decompile, unconfirmed on hardware)
- `fcn.4201ea04` 0x4201ea04 (924 bytes) is the function that turns a pot move into a parameter change. Call (Xtensa windowed regs, read from `fcn.4201f0f0` disassembly): a10 = section/group object, a11 = pot index (0-14), a12 = UI parameter index, a13 = raw pot value (0-511), stack arg at sp+0 = optional second parameter index (-1 = none).
- What it does: keeps per-pot state at ctx+0x270 + pot*0x14; ignores moves under 0x1f counts unless 100 ms passed; converts value/511 to 0-1; optional pickup/soft-takeover (setting read via `fcn.4204f8d0(0x3fccfd14, 8)`) with a bipolar centre snap of +-0.03 (`fcn.42018e80` = is-bipolar); quantises stepped params (`fcn.42018f08` / `fcn.42018f9c` give the step count); then builds a 24-byte event {param index, ?, 6, 0, value = float * 1e6 as int} and sends it with `fcn.4201d010(ui, &event)`. If the second index is >= 0 it sends a second event with the same value.
- UI parameter index: if index < 0x150 the part offset is added (current part at ui+0x9a0, times 0x54). So the index space is per-part with stride 0x54 (descriptor idx - 25 for sound params, plus 1 offset for some; WaveEnv A-R = 0x13-0x16, FiltEnv A-R = 0x20-0x23, GlideRate = 0x10, Pan = 0x35, Volume = 0x37).
- Who calls it (parameter index each passes): Amp section `AmpSection__onEvent` `fcn.4202006c` -> 55 (Volume); `fcn.42020140` -> 53 (Pan); `fcn.4202049c` -> 16 (Glide) and 1 and 8; Wave section `fcn.4201f6a0` -> 3, 10, 5, ...; Filter section `fcn.4201fd4c` -> pot 3 and 4 plus a computed index (+41); knob group `fcn.4201f0f0` -> index read from obj+0x24/0x26/0x28/0x2a chosen by the slot at obj+0x2c, pot id at obj+0x2e.
- Event type 7 from the engine back to the UI sets the UI-side parameter store (`fcn.4204f8f0(ui+0x9a4, index, value*1e-6)`), so the UI mirror updates from the engine.
- Remap idea (not built): a trampoline hooked at the start of `fcn.4201ea04` that, when an "envelope mode" is active, rewrites a12 for the Glide, Wave, Filter and Amp pot indices to the A, D, S, R indices of the selected envelope (Wave 0x13, Amp 0x1b, Filter 0x20 as bases, if the Amp base is right), with Glide as shift for Attack. Needs: the four pot indices (read from each section's code), a free flash area, a way to select the envelope and the shift, and a checksum and SHA-256 fix on the image.

### Knob map (session 2)
- See KNOBS.md (all 15 pots mapped to parameters, with and without Shift, plus helper functions) and analysis/ui_params.txt (firmware UI parameter table with names, 84 per-part entries and 225 global entries).
- Corrects earlier notes: UI indices are the firmware's own table, not descriptor index minus 25. Envelope groups are Wave 0x13-0x16, Filter 0x25-0x28, Amp 0x20-0x23. Pots: Glide = 1, Wave envelope = 5, Filter envelope = 6, Amp envelope = 9.

### Not mapped yet
- Arp engine (for latch), envelope DSP, display/page code, storage format of sounds.

## Next steps
1. Finish mapping: arp, knob-to-parameter routing, user wavetable slots.
2. Install Espressif's xtensa-esp32s3-elf toolchain (blocked in the cloud workspace; works on your PC/WSL).
3. Patch builder: compile C mods, place in free flash, patch call sites, fix checksum/SHA-256, wrap as UF2
   (family 0x12e6002f, same addresses).
4. Harmless test mod (e.g. change a menu string), flash, confirm the SHIFT bootloader can restore stock.

## Wavetable findings (round 2)
- Step to table: step s -> table s if s < 32 else s + 0x18. Confirmed in UI formatter fcn.420177dc and engine voice code fcn.4037ac44 (params 4 and 11) and fcn.4037b634. Renderer clamps table number to 0x57.
- Cache: MW1WavetableCache allocated in PSRAM by fcn.4201682c, global pointer 0x3fcbdcc0. Region is zeroed at init. User table defs (cache+0x5627a, 24 x 0x7a bytes) and user waves (cache+0x56e02, 122 x 64 bytes) start empty.
- Init loop fcn.4202c500 does not build user tables 0x20-0x37. Nothing found that fills them.
- Storage: NVS namespace "data". Keys "sounds", "sound-%03i", "globals" (default). Storage::load 0x420278a0, save 0x42027938. Callers of save: 0x420193ec, 0x42027aac, 0x42027bc4.
- Sysex sound dump carries parameter tags only, no wave data.
- Unconfirmed: exact cache size (ctor 0x42025074 sizeof log), partition table and free NVS space (not in UF2).
- Patch outline (untested): descriptors idx 29 and 36 step count 64 -> 88; remove +0x18 skip in the three sites above; call buildTable (0x4202b6c4) for t in 0x20-0x37 after filling defs; set voice reload flags (+0xcb/+0x15f).

## Arp and sequencer (round 3, from disassembly, untested on hardware)
Note path
- Engine event type 0 = note on, type 1 = note off, handled in fcn.4200d7f4. They call vtable +0x14 / +0x18 on the object whose pointer is stored at 0x3fca5010 (probably the note processor singleton 0x3fca5828; not checked). Type 9 -> vtable +0x4c, with value forced to 1 if nonzero (likely sustain pedal; goes to stub forwarders, so the arp probably never sees the pedal; unconfirmed).
- Note processor singleton 0x3fca5828 (vtable 0x3c070110, +4 = secondary vtable 0x3c07018c, +8 = engine object 0x3fca7508). Sub-objects: clock at obj+0x10, held-key table at obj+0xa0 (128 bytes, one per note), ARP OBJECT at obj+0x728.
- vtable +0x10 fcn.42013df0 = setup: calls arp setup fcn.42017718(arp = obj+0x728, clock = obj+0x10, held = obj+0xa0, sink = obj+4).
- vtable +0x60 fcn.42013e30 = per-block tick: calls arp tick fcn.42017670(obj+0x728), then clock tick fcn.42016bb0, then forwards to voices.
Arp object (A = obj+0x728): A+0 clock, A+4 held table pointer, A+8 sink (obj+4). Arp generator state at A+0x14 (vtable ptr 0x3c071f34 at A+0xc), sequencer state at A+0x54. Both are given the held table pointer (fcn.4204ecf8 sets A+0x20 area +4/+0x2c; fcn.4204ed40 sets A+0x54+8). A+0x368 = pattern params block. Step runner at A+0x13d0. Mode at A+0x142c (= obj+0x1b54), previous mode A+0x1430, running A+0x1434 (= obj+0x1b5c), runner active A+0x1435, "no keys" flag A+0x14b8, arp sounding-notes table A+0x1438 (128 bytes).
- fcn.420175d8: on mode change, rebinds the runner to the generator for mode 0 / 1 / 2 (configs at A+12, A+20, A+84).
- fcn.42017570: if all 32 words of the held table are zero -> stop runner (all notes off to sink); else if runner inactive -> start it (fcn.42018810). Called each tick when mode is 1 or 2 and the "new note" flag obj+0x721 is set.
- fcn.42018858: runner step. Asks generator vtable +0x18 for the next notes (up to 4), sends note on via fcn.42017494 and note off via fcn.420174f0 to the sink.
- fcn.42017b0c: arp generator step; algorithm index 0/3 up, 1/4/5 down, 6 random (reading only partly checked). Reads the held table through fcn.4201787c / 4201799c / 42017900.
Held table writers: noteOn fcn.42014528 (sets obj+0xa0+note = 1, also velocity, timestamp, flag obj+0x721), noteOff fcn.42013e90 (clears at two sites: 0x42013ed9 chord notes loop, 0x42013f1c main note). While running flag obj+0x1b5c is set, note events are not forwarded to the voices; the arp plays them instead.
Latch design (untested)
- Latch = keep held[note] set after key release while mode is 1 or 2. Patch the two clear sites in noteOff to skip when latch is on and obj+0x1b54 != 0.
- noteOn: when latch is on and no key is physically down, clear the 128-byte table first (loop already exists at 0x42014581), then set the new note.
- Needs a physical-key record (a counter or 128-byte bitmap in spare RAM; location not chosen), and a clear of unheld notes when latch is switched off.
- Sequencer uses the same table, so the same patch should latch both (inferred).
- No latch parameter or button exists. Toggle control still to decide; needs the button map.

- Button and encoder map: see BUTTONS.md (round 4).

## Arp and sequencer (round 4) - roadmap item 1, partly done

Engine parameter ids (setParam fcn.42013f40, note processor obj, id = event param):
- 0 ArpSeqMode -> obj+0x1b54 (0 Off, 1 Arp, 2 Seq). Turning on calls start fcn.4201763c, turning off calls stop fcn.42017534. Changing from Off first calls vtable +0x50 (all notes off) and clears held table obj+0xa0.
- 1 ArpAlgo -> obj+0x758. 2 ArpOctaves -> obj+0x760 (stored as value+1). 3 ArpReset -> obj+0x748 (+ flag obj+0x74c = 1). 4 RhythmPattern -> obj+0xad4 (0-15). 5 BPM (float) -> obj+0x10. 6 RelGate (float) -> obj+0xac4. 7 StepLen (index) -> obj+0xac8. 8 Swing (float) -> obj+0xacc.
- 9 SeqLen -> obj+0xa09. 10 SeqDir -> obj+0xa08. 11 SeqReset -> obj+0xa0c. 12 SeqScale -> obj+0xa0a. 13 SeqRoot -> obj+0xa0b. (the sequencer object base is obj+0x77c, byte params at base+0x28c..0x290)
- 0x10 ChordMode (UI 0x22b) -> obj+0x1be9, 0x11 ChordType (0x22c) -> obj+0x1bf4, 0x12 ChordScaledType (0x22d) -> obj+0x1bf5, 0x13 ScaleType (0x22e) -> obj+0x1bf6, 0x14 Flavour (0x22f, not handled in the note processor). Setting 0x10 or 0x13 rebuilds the scale mask with fcn.42017348 (table obj+0x1bf8, scale = ScaleType, root = ChordMode-2 as coded) and the 128-entry quantize map at obj+0x1c60. The Chord feature therefore lives in the note processor note-on path.
- 0x15 + 4*step + k, step 0-31: k=0 pitch byte obj+0x789+0x14*step; k=1 velocity float obj+0x78c+0x14*step; k=2,3 only stored for step 0 (obj+0x794, 0x798). Sa/Sb of steps 1-31 are ignored (confirmed in disassembly at 0x42014345-0x42014393). Step record is 0x14 bytes: byte +0 (obj+0x788) step type, +1 pitch, +4 velocity float, +8 float (not set by these params), +0xc and +0x10 mod floats. Step type byte writer not yet found.

Held-key object (shared by arp and sequencer, pointer at A+4): byte flags [128] at +0x00, velocity float [128] at +0x80, 64-bit press timestamp [128] at +0x280 (8 bytes each). Helpers (confirmed by reading the code): fcn.4204e960 lowest note, fcn.4204e990 highest note, fcn.4204e9c8 oldest pressed, fcn.4204ea4c newest pressed, fcn.4204ebd8 lowest velocity, fcn.4204ec4c highest velocity. The older "obj+0xa0 128 bytes" is just the flag part.

Arp generator (fcn.42017b0c):
- Algo values: 0 Up, 1 Down, 2 Up-Down, 3 Up|Down (repeats end notes), 4 Down-Up, 5 Down|Up, 6 Random. Matches manual order. Octave counter at gen+0x28, octave count at gen+0x24, direction at gen+0x3c. Octave offset is added as 12 per octave, wrapped down when above 127.
- Random: random note via fcn.42017ab8 (walks up to 19 steps over the held notes) and random octave = rand() % octaves.
- Note order type = int(float at gen+8), default 2.0 -> 2. Types: 0 oldest to newest pressed, 1 newest to oldest, 2 lowest to highest pitch (default), 3 highest to lowest, 4 lowest to highest velocity, 5 highest to lowest velocity, 6 chord (outputs up to 4 held notes at once, random-ish start). The float is not in the UI parameter list, so what writes it (probably a fixed value or another setter) is not yet found; the panel arp uses the default pitch order.
- A float at gen+0x14: 1 = velocity of oldest held key, 2 = velocity of newest held key, else the velocity of the note played.

Rhythm clock object (fcn.42017f08/f78/ff8/18098/18104/18260/181e8):
- StepLen index 0-16 -> beats from table at 0x3c073118 (doubles): 4, 3, 2, 1.5, 1, 0.75, 0.667, 0.5, 0.375, 0.333, 0.25, 0.1875, 0.1667, 0.125, 0.09375, 0.0833, 0.0625. Reading as beat counts is inferred from the values and from the manual (1/16 default).
- Swing: s = 0.05 + 0.9*x (x = swing 0..1 plus modulation, clamped), odd steps use 1-s, step time = steplength * 2 * s. Display value is 100*s (50 = straight, 5 to 95 range). Matches manual.
- Gate: note-off time = step start + RelGate * step duration. A step flagged tie (2) skips the note-off.
- Rhythm patterns: 16 patterns x 32 steps x 8 bytes (flag byte: 0 rest, 1 note, 2 tie; accent float, all 1.0) at clock+0x5c. Pattern 0 = every step. Patterns 1-15 come from the table at 0x3c072d54 (16-step masks repeated over 32 steps):
  1 x.xxx.xxx.xxx.xx | 2 x.x.x..xx.x.x..x | 3 x.x.x.xxx.x.x.xx | 4 x.xxx.x.x.xxx.x.
  5 x.x.xx.xx.x.xx.x | 6 xx.x.xx.xx.x.xx. | 7 x.x.x.x.xx.x.x.x | 8 x.x.x.xx.x.xx.x.
  9 xxx.xxx.xxx.xxx. | 10 xx.xx.xx.xx.xxx. | 11 xx.xx.xx.xx.x.x. | 12 xx.xx.x.xx.xx.x.
  13 x.x.x.x.xx.x.xxx | 14 x..x..x..x..x..x | 15 x.x.x.x.x..xx.x.
  (x = note, . = rest; no accents or ties in the factory tables)
- Runner fcn.42018858 per tick: update clock, release notes whose gate has ended, ask the generator (vtable +0x18) for the next notes/velocities/tie flag, send note-on via fcn.42017494 and note-off via fcn.420174f0, count steps (vtable +0x1c) and reset the generator at the end of the cycle (vtable +0xc).

Sequencer step function (fcn.42018640, vtable 0x3c0731a8):
- Base note = newest held key (fcn.4204ea4c). Each step plays base + stored pitch byte - 48 (stored 48 = "+0"), then moved to the nearest note of the chosen scale/root (scale table at 0x3c071bcc / 0x3c071e0c). New pitch above 127 wraps down by octaves.
- Step type byte: not 1 = note, 2 = tie (inferred from how the flags are output).
- SeqLen is modulatable: length = value + 1 + int(mod * 31), clamped to 1-32 (pointer to a mod float at seq+0x294).
- SeqDir 0 forward, 1 backward, 2 ping-pong without repeating end steps, 3 ping-pong with repeated end steps, 4 forward once then stop. Matches manual.
- SeqReset (fcn.420182c8): 0 Start (step 0, or last step if backward), 1 None (keep position), 2 Random (rand() % length). Matches manual.
- Two extra values per step (+0x18, +0x1c, scaled to -1..+1) are published at seq+0x308 and seq+0x30c, probably as modulation sources. Unconfirmed.

Open points of round 4, now resolved:

1. Step type byte (A+0x60+0x14*step, engine obj+0x788+0x14*step): nothing writes it. No setParam id, no editor and no record code touches it; only the zeroed object memory sets it. So every step is a note (type 0) and the sequencer has no rest or tie of its own. Rests come only from the rhythm Pattern (flag 0). In the step function type 1 would mean rest (note count 0) and type 2 tie, but those values cannot occur (confirmed by searching all accesses of the object at offset 0x788 and the 0x700-based accesses in the note processor code).

2. Record (UI page 5, object UI+0x548, vtable 0x3c0736d0):
- Start (list action -9, fcn.420228c0): sets the page recording flag (+0x35 = 1), step counter (+0x36 = 0), writes ArpSeqMode (UI 0x19b) = 0 (Off), and makes the page the modal event target (fcn.4201d328, so it receives MIDI note events first).
- Each MIDI note-on (status 0x90-0x9f, any velocity including 0, handler fcn.42022bc0): step 0 stores the reference note (page+0x38). Pitch value = (note - reference) + 0x30, moved by whole octaves into 0..0x60, written to Sp (UI 0x1a9 + 4*step). Velocity = vel/127 written to Sv (UI 0x1aa + 4*step). SeqLen (UI 0x1a4) is written with the step index (so length = steps recorded). Counter +1. At 32 steps recording stops by itself.
- Stop (fcn.42022908, Shift+Arp/Seq or any press while recording): clears the flag and modal, writes ArpSeqMode = 2 (Seq).
- Parameter writes from the UI use events {param id, value*1000000 as int, type 6}, sent with fcn.4201d010.
- Only pitch and velocity are recorded, no timing and no rests; it ignores note-offs.

3. Step editor (page 6, UI+0x584, vtable 0x3c073758): fields +0x24 (0 = value mode, 1 = field-select mode, toggled by pressing the Select encoder, button id 10), +0x25 selected field (0 step, 1 pitch, 2 velocity), +0x26 current step (0-31). Encoder 0 turn in value mode: field 0 moves the step cursor (0-31), fields 1 and 2 edit the value through the generic encoder writer fcn.4201eda0 (I did not trace how it picks the Sp/Sv param of the current step, assumed). In field-select mode turning changes the field. Draw function fcn.42022d98: bars for up to 16 visible steps (window follows the cursor), pitch shown as (byte - 0x30) clamped to +/-22, velocity shown as percent (value * 100). Leave with Shift+Arp/Seq (manual).

4. Arp note order: it is int(float at arp-generator +8), default 2.0 set in constructor fcn.42017e28, which is object offset 0x744 of the note processor. No code in the note processor writes offset 0x744 (all accesses through the 0x700-based address form were checked), and setParam has no id for it. So the order is fixed at type 2 (lowest to highest pitch) as far as this search shows. The chord output (type 6) is therefore not reachable. Possible hook for a future feature.

Other facts found while resolving these:
- Layout of the arp object A (note processor obj+0x728): A+0x0 clock, +0x4 held-key object pointer, +0x8 sink, +0xc mode-0 pass-through generator, +0x14 arp generator, +0x54 sequencer object, +0x60..+0x2e0 sequencer steps (32 x 0x14), +0x2e8 pointer to A+0x364 (always 0.0, so the sequence length modulation is never active), +0x364 zero float, +0x368 rhythm clock (patterns at +0x5c inside it), +0x13d0 runner, +0x142c mode, +0x1430 previous mode, +0x1434 running, +0x1435 runner active, +0x1436 flag, +0x1438 sounding table (128 bytes: 100 after note-on, 2 after note-off).
- Mode to generator: 0 -> A+0xc, 1 -> A+0x14 (arp), 2 -> A+0x54 (sequencer), selected in fcn.420175d8.
- The held-key object base is obj+0xa0; its +0x680 byte is passed to note-off (probably the sustain state, unconfirmed) and +0x681 is the new-note flag.
- The modulation pointers of the rhythm clock for swing and gate (+0x48, +0x4c) are null, so swing and gate are not modulated.

## Engine events and note path (round 5) - roadmap item 2

Objects: note processor singleton 0x3fca5828 (vtable 0x3c070110, secondary "sink" vtable 0x3c07018c at obj+4) wraps the sound engine at 0x3fca7508 (vtable 0x3c070218, at obj+8). The engine holds 4 part objects (stride 0xcb8, start 0x3fca7510): these are layers A-D. Each part has 8 voice slots (stride 0x70, 8 entries) and its own voice class vtable 0x3c070268 (setParam at +0x40 = 0x4037ac44).

MIDI input (fcn.42019bdc, channel messages; realtime 0xf0+ goes to fcn.4201beac for Start/Stop/Continue/SongPos/Reset):
- UserInterface__processMidiInput runs first (MIDI learn). Channel is converted to a part index relative to the receive channel; channels outside the accepted range are dropped. With LayerMode 4 (Multi) and LayerAlloc 3, 4 channels are accepted, otherwise 1.
- Engine events are 0x18 bytes: [type, a, b, c, d, e], queued with Engine__sendEngineEvent(0x3fca3974), then dispatched by fcn.4200d7f4 to the note processor object (pointer 0x3fca5010).
- 0x90 with velocity > 0 -> type 0 note on [note, vel, part]. 0x90 with velocity 0 and 0x80 -> type 1 note off. 0xa0 -> type 5. 0xb0 -> fcn.420199d0 (also type 3). 0xc0 program change: bank*128 + program, stored at +0x1af4 if below 0x168 (360 presets). 0xd0 -> type 4. 0xe0 -> type 2 with 14-bit value (data2<<7 | data1).
- CC handler fcn.420199d0: CC 1 -> UI param 0x230 (mod wheel); CC 0x20 -> bank LSB (+0x1af6); CC 0x7c/0x7d omni off/on (fcn.42029120). Other CCs go through the user CC map (u16 UI param index per CC 0-0x77 at +0xc6c, the CC Mappings page) and set that UI param. Every CC is also sent to the engine as type 3.

Dispatcher (fcn.4200d7f4), event type -> note processor vtable offset:
- 0 note on -> +0x14 (fcn.42014528, held table, arp/seq aware)
- 1 note off -> +0x18 (fcn.42013e90, same)
- 2 pitch bend -> +0x28 -> engine fcn.42015198 -> part fcn.42014788 (value -> -1..+1, per-note index below 0x80 else global)
- 3 control change -> +0x1c -> engine fcn.42015048 -> part fcn.4201486c (CC 0x40 sustain, 1, 2, 4, 0xb smoothed, 0x4a per-note, 0x78 all sound off, 0x7b all notes off)
- 4 channel pressure -> +0x20 -> engine fcn.420150c0 -> part fcn.420147f4
- 5 poly aftertouch -> +0x24 -> engine fcn.4201512c -> part fcn.42014738
- 6 set parameter -> lookup fcn.42018a88(id), then +0x40 (note processor setParam fcn.42013f40, which then forwards to the engine setParam fcn.420157e0)
- 7, 8, 9 -> +0x44, +0x48, +0x4c: engine writes part+0x3c0 (int), part+0x3c4 (float, event value * 0.01), part+0x3c8 (bool) on all 4 parts. By their shape they are probably Transpose, Master tune in cents and MPE status, but no sender was checked. Unconfirmed.
- Only types 0, 1 and 6 are handled inside the note processor. All other types are plain forwarders (0x4204e4f8 etc.) to the engine, so the arp and sequencer never see them.

Sustain pedal (confirmed by reading code): CC 64 -> type 3 -> forwarder -> engine -> part fcn.4201486c -> part method +0x2c -> fcn.4204e728 sets part+0x398 (pedal down) and, on release, sends note-off for every note marked 2 in the table at part+0x62c. Note-off in the part (fcn.40380afc) marks a note as 2 (held by pedal) instead of releasing when pedal is down. So the pedal acts at the voice level and the arp cannot see it. With the arp running, pedal down makes every arp note ring until the pedal is released. Latch cannot be built from CC 64 without changing the note processor.

Note routing in the engine (note on fcn.42014cec; selector floats at engine+0x3324 = LayerMode (engine param 0xe, UI 0x229) and engine+0x3328 = LayerAlloc (engine param 0xf, UI 0x22a)): when LayerMode is below 4 or LayerAlloc is 0 the note goes to all active parts; with LayerMode 4 (Multi) LayerAlloc 3 sends the note to the part given by the event, 1 rotates through parts in order, 2 picks a random part. A part has 8 voices (polyphony 1-8 via part+0x392, setter fcn.4204e824), round robin by index (part+0x390), with a note-state table at part+0x62c (0 free, 1 on, 2 held by pedal). Unconfirmed: the display names of the LayerAlloc values.

Engine vtable 0x3c070218 (for reference): +0x10 init/load defaults (ParamMgr__loadDefaults fcn.4201584c, 0x95 global params, 4 parts), +0x14 note on, +0x18 note off, +0x1c CC, +0x20 channel pressure, +0x24 poly pressure, +0x28 pitch bend, +0x2c sustain (part fcn.4204e728), +0x34 active voice count, +0x38 set polyphony, +0x3c read param, +0x40 setParam, +0x44/+0x48/+0x4c types 7/8/9, +0x50 all notes off, +0x54 all sound off.

## Sound engine, envelopes (round 6) - roadmap item 3, part 1

Parameter indexing: the engine ids (eng=0x95 and up in ui_params.txt) are part-relative index k = eng - 0x95, which is NOT the UI index order. The per-part float array (one per part A-D, pointer at voice+0x5b8) is indexed by k. The part setParam (fcn.4037ac44) handles only derived values (volume, pan, LFO, mod matrix, filter amounts, table numbers). It does not touch envelope parameters.

Envelope parameters (all plain floats 0..1, k and byte offset in the per-part array):
- Wave env: Attack k=0x13 (+0x4c), Decay 0x14 (+0x50), Sustain 0x15 (+0x54), Release 0x16 (+0x58). Wave1 amount 0x17, Wave1 vel amount 0x18, Wave2 amount 0x19, Wave2 vel amount 0x1a.
- Amp env: A 0x1b (+0x6c), D 0x1c (+0x70), S 0x1d (+0x74), R 0x1e (+0x78), velocity 0x1f (+0x7c).
- Filter env: A 0x20 (+0x80), D 0x21 (+0x84), S 0x22 (+0x88), R 0x23 (+0x8c), amount 0x24 (+0x90, stored as (v-0.5)*2 at voice+0x4c4), velocity amount 0x25 (+0x94, same form at voice+0x4c8).
- There is no delay, hold or curve parameter for any envelope. There are exactly 3 ADSR envelopes.

Where they run: the voice render function fcn.4037bfb0 (IRAM, size 10092). Each block the 12 values are read live from the array, added to a mod-matrix offset and clamped to 0..1, then copied into voice fields: wave A/D/S/R at voice+0x500/504/508/50c, amp at +0x520/524/528/52c, filter at +0x540/544/548/54c. The mod-matrix offsets sit at voice+0x784 + 8*n (wave 0x784, 0x78c, 0x794, 0x79c; filter 0x7a4, 0x7ac, 0x7b4, 0x7bc; amp 0x7c4, 0x7cc, 0x7d4, 0x7dc), so the mod matrix can modulate envelope times and sustain.

Because the values are read live, writing the parameter is enough: nothing else needs updating when a knob changes an envelope value. Any new control that sets these parameters through the normal parameter write will be heard immediately, also on notes already sounding.

Stage machine (identical for the three envelopes; level fields: wave +0x4f8, amp +0x518, filter +0x538; stage fields: wave +0x510, amp +0x530, filter +0x550; counters +0x514, +0x534, +0x554; delay thresholds +0x4fc, +0x51c, +0x53c):
- It runs once per 8 render blocks, unless a voice flag at +0x5c0 is set.
- Stage 0 idle. Stage 1 delay: level 0 until the counter reaches the delay threshold (no parameter sets it, so it is passed at once; set elsewhere, not traced). Stage 2 attack: level += (1 - level) * c[A*127], finished when level*32767 >> 8 reaches 0x7f (about 0.996) or when A is 0, then stage 3. Stage 3 decay: level += (S - level) * c[D*127] until level is at or below S, or D is 0; then level = S and stage 4. Stage 4 sustain: level = S (follows S live). Stage 5 release: level -= level * c[R*127] until below 0.0001 or R is 0, then level 0 and stage 6 (finished).
- So every stage is a one-pole exponential. Attack is not linear: it approaches 1.0 asymptotically and is cut at about 0.996.
- c[i] is a 128-entry coefficient table in RAM at 0x3fccde98, filled once at start (in fcn.420162d0) from a ROM table at 0x3c070340 (128 integers): c[i] = ROM[i] / 16 / 32767, but 1.0 when ROM[i] is 32767. ROM values: i=0 and 1: 32767 (instant), 2: 30500, 3: 28500, 4: 26500, 5: 24800, then falling about 7 percent per step to 20 at i=108, 15 at i=112, 9 at i=120, 5 at i=127. The index is int(value*127), so each knob has about 127 steps, and the first two steps are instant.
- Time in seconds is not derived: the block size and the sample rate were not found in the code, and the manual has no sample rate. Unconfirmed.

Consequences for the planned envelope knobs: no engine change is needed, only UI side handlers that write the existing parameters (already found in KNOBS.md). For a "selectable envelope" only a choice of which three parameter groups the knobs write to is needed.

Still open in item 3: oscillator and wavetable playback, filter (types, drive, noise, ring mod), LFOs, mod matrix sources and destinations, velocity paths, voice allocation, and the Flavour logic.

## Sound engine, oscillators and wavetable playback (round 7) - roadmap item 3, part 2

All in the voice render function fcn.4037bfb0 (IRAM). Per-part parameter indices k 0-0x12 equal the UI indices: 0 Osc1Semitone, 1 Osc1Detune, 2 Osc1Keytrack, 3 Osc1Volume, 4 Wtb1Table, 5 Wtb1Pos, 6 Wtb1LimitWT, 7-0xd same for osc 2, 0xe DirtType, 0xf NoiseVolume, 0x10 GlideRate, 0x11 GlideMode, 0x12 PitchBendRange.

Wave position (control rate, per block): position = base pos (voice+0x37c, from Wtb1Pos, osc 2 from its own) + (extra + velocity part) * wave envelope level + mod-matrix offset * 63. Result is limited to 0..63, or to 0..60 when LimitWT is on (voice+0x3cd for osc 2). The integer part becomes the wave index (voice+0xc5 osc 1, +0x159 osc 2). There is no crossfade between waves: the position is an integer 0..63 and a change is picked up only when the oscillator phase wraps (flag voice+0xcb for osc 1, +0x15f for osc 2). So a wavetable sweep is stepped at cycle boundaries.

Wave data lookup (confirmed in the decompile at render.c lines 4901-5400):
- Table number (voice+200 osc 1, +0x15c osc 2) is limited to 0..0x57. Wave w of the table is built from cache entry t*0x3d + w for w < 61. Waves 61, 62, 63 come from table 0x58 entries 0, 1, 2 (the generic triangle, square, sawtooth of the manual). So the cache holds 89 tables (0-0x58) of 61 waves, each wave 64 bytes at cache + 4 + (t*61 + w)*64, unsigned 8-bit centred on 0x80.
- Each stored wave is only half a cycle. It is expanded to 128 signed samples: samples 0-63 = byte - 0x80; samples 64-127 = 127 - byte[63 - k], i.e. the first half reversed and inverted. So every playable waveform satisfies x[127-n] = -x[n] - 1 (odd symmetry about the middle of the cycle). Any waveform is therefore a sum of sine terms only, with no cosine-phase content. (Inferred from the code, not tested. A saw and a square fit this form.)
- The expanded wave is copied into the voice (osc 1 at voice+0x3c, 128 bytes; osc 2 at voice+0xd0) when the wave or table changes. The decoder first rebuilds all 64 waves of the table into a per-oscillator buffer (obj at voice+0x38 for osc 1, voice+0xcc for osc 2: [0] cache pointer, [1] current table number, then 64 waves of 0x80 bytes from +8).
- Playback: 32-bit phase accumulator (osc 1 at voice+0xbc, increment +0xc0; osc 2 at +0x150, increment +0x154), sample = wave[phase >> 25], 7 bit index, no interpolation between samples.
- Mix per sample: out = vol2 * s2 + vol1 * s1 + ringAmount * s1 * s2 / 51200 (constant 1.953e-5). The ring mod (Dirt type Ring Mod) is that product term. Result goes into a 128-sample circular buffer at voice+0x164.

Pitch (per block): osc 1 = ((voiceNote + Semitone) - 24 + (Detune - 0.5)*2 + note-or-60 + (mod offsets)*24) * 256 in 8.8 fixed point; osc 2 the same without the extra voice term. Keytrack off replaces the note by 60. The fixed-point pitch is folded into range by subtracting 0xc00 (12 semitones) and converted to a phase increment by interpolating a ROM table at 0x3c070d40, times 0x800 (offset constant 0x93a). Detune is plus or minus 1 semitone, matching the manual.

Consequences for custom wavetables (important):
- A custom wave is 64 bytes and is expanded by the odd-symmetry rule above. To import a recorded cycle, the Windows tool should take its harmonic magnitudes (FFT) and rebuild a sine-phase version of the wave. Phase is lost, the spectrum is kept. Shapes with cosine-phase content cannot be represented without changing the decoder code.
- Allowing true 128-sample waves would need a patch of this decode loop and a larger cache (stride 0xf40 = 61*64 would become 61*128), more RAM and bigger tool output. Not designed.
- The table clamp 0x57 and the 61 + 3 layout stay as found earlier; user tables 0x20-0x37 fit inside it.
- Table or wave changes happen at the phase wrap of the oscillator, which also means a new table is picked up by a sounding note on its next cycle.

Unconfirmed in this part: block size (loop count per call), Glide and Flavour code, the +0x30 voice term in the osc 1 pitch, keytrack scaling.

## Sound engine, filter and drive (round 8)

Where: voice render fcn.4037bfb0 (filter block near its end), voice setParam (ALL.c ~3000-3160), voice constructor fcn.4201597c (constants), drive core fcn.42011584 / wrapper fcn.42011768. Processing is in blocks of 16 samples.

Parameters (per part, k = eng-0x95, byte offset = 4*k):
- FilterCutoff k=0x30 (+0xc0), FilterReso k=0x31 (+0xc4), FilterType k=0x32 (+0xc8), FilterKeytrack k=0x33 (+0xcc), DriveType k=0x34 (+0xd0), k=0x35 (+0xd4) ramped value at voice+0x4e0 (not identified).
- Filter env amount k=0x24, filter env velocity amount k=0x25 (see envelopes).

Cutoff (0..1 units) per block:
cutoff = ramped_param(voice+0x4b0) + voice+0x2c (random drift, small) + (envAmt(+0x4c4) + velAmt(+0x4c8)*velocity(+0x10)) * filterEnvLevel(+0x538) + keytrack(+0x4c0)*(note(+0x1c)-60) + modmatrix offset (+0x754), clamped 0..1.
- Resonance = param + modmatrix offset (+0x75c), clamped 0..1.
- Param change is not applied instantly: voice+0x4b0 ramps linearly to the new value over voice+0x4be samples (step stored at +0x4b8).
- Keytrack: (param-0.5)*0.0625 cutoff units per semitone around note 60 (param 0.5 = none).
- envAmt = (param-0.5)*2, so the amount is bipolar.

Filter type (voice+0x4ac = int(param+0.05)), 5 values 0..4. UI names for these 5 values were not found as strings, so the names below are descriptions of the DSP only:
- 0 and 1: one structure (coefficient set at voice+0x3e4..0x43c, four cascaded one-pole stages with feedback and soft saturation). Type 1 uses resonance x0.8. Both limit cutoff to 0.9 when resonance > 0.62 (type 1 tests resonance x0.8 against 0.62). Output gain after filter 0.75.
- 2 and 3: second structure (coefficient set at voice+0x460..0x4a8, four stages). Resonance multiplied by 2.4 (type 3 first x0.85). Output gain after filter 1.2.
- 4: no resonance filter. Cutoff is used as drive amount: object at voice+0x3d4 gets amount = cutoff*0.5 (at +0x3d8), and output gain (1-0.7*cutoff)^2 (at +0x3d0). Resonance dial is shown as DriveType for this type (BUTTONS.md).
- Any other value: passes through unfiltered (gain 1.0).
Cutoff to coefficient (types 0,1): g = 0.000582 * 2^(1.6667 + 10.4167*cutoff). Types 2,3: g = 0.000582 * 2^(12.0833*cutoff). (Constants set in constructor: 0x404=10.4167, 0x408=1.6667, 0x40c=0.000582, 0x460=12.0833, 0x464=0, 0x468=0.000582.) The 2^x is done with an integer shift plus polynomial (1 + 0.6931472 f + 0.2402265 f^2).
If g is read as tan(pi*fc/44100) (assumption, not verified), cutoff 0 / 0.5 / 1 gives about: types 0,1: 26 Hz / 958 Hz / 16.8 kHz; types 2,3: 8 Hz / 538 Hz / 16.8 kHz.

Drive core fcn.42011584 (DriveType k=0x34, value clamped 0..4 stored at drive object +0xc; object+0 = 0.2 output scale, +4 = amount, +8 = previous input sample):
- gain = amount^2 * 128, input is the average of current and previous sample (types 0,1,2,4).
- Index 0 (default path): symmetric parabolic soft clip (2x - x^2 for 0..1, mirrored, hard limit at +-1).
- Index 1: input biased +0.707, piecewise: below 0 outputs -scale, above 1 outputs +scale, between a parabola.
- Index 2: biased +0.707, squared minus 0.5, limited to 0.5, then x - x*|x| shape, x4.
- Index 3: no averaging; gain*x limited at -1, then x^2 + 2x, limited to 1.
- Index 4: fractional part (wrap) of gain*x mapped to -1..1, then x - |x|*x (folding style).
- Wrapper fcn.42011768 crossfades with the dry signal when amount < 0.125 (cutoff < 0.25).
- Names by index follow the UI list order PNP, Tube, PickUp, Diode, Crunch. That order is the string order in the firmware, not confirmed to equal the index order.

Noise: NoiseVolume k=0xf is squared, then ramped (target voice+0x3b0, step +0x3b4, current +0x3ac) and added in the output stage clamp section. DirtType (k=0xe) and the noise generators (Static, Crackle, Geiger, Click, Burst 1-3, Ring Mod) were NOT found in the voice render; they are likely at part or engine level. Open.

Output stage after the filter: amp env level (+0x518) times (+0x4d0 level + modmatrix +0x764), then a 16-sample linear ramp from the previous block value (+0x558, +0x55c) to the new value, written to the left and right buffers (in_a3 and in_a4, so panning/level is part of this ramp). Not fully traced.

Other findings this round:
- 44100.0 appears in the voice start code (glide time: samples = distance / (rate^3 * 44100 * 3)). Sample rate 44.1 kHz is now supported by the code, block size 16 samples.
- Block of 16 samples at 44.1 kHz = 0.363 ms per block. (Earlier claim 'envelopes advance every 8th block, about 2.9 ms' was wrong; see round 10: ticks are about 49 samples and the envelope interval is unconfirmed.)
- The firmware already contains ms and seconds formatters: "%.0f ms", "%.1f ms", "%.3f ms", "%.3f s" (used by compressor/effects). Relevant to improvement 4.

Open points:
- UI names for the 5 filter type values.
- Whether coefficient g is tan-based (frequency estimates above depend on it).
- Meaning of k=0x35 (+0xd4) and the flag deciding between "update coefficients" and "reuse coefficients" paths in the filter section.
- DirtType and noise generator code location.

## Sound engine, LFOs and mod matrix (round 9)

CORRECTION to round 8 timing note: "envelopes advance every 8th block (about 2.9 ms)" is NOT confirmed. The voice render only runs its slow section (LFO products, envelopes) when the global tick counter (engine+0x402c) has advanced. Global tick function fcn.4037e71c (vtable entry at 0x3c071158, caller not yet found) does: counter(+0x4024) += 16 per call; when counter >= threshold(+0x4028): tick count +1, threshold += 49 (random +-32 jitter when part setting at +0x50 > 0.25). If it is called once per 16-sample block, a tick is about 49 samples (1.11 ms) and an envelope step (every 8th tick) about 8.9 ms. If not, the figure differs. Envelope time-in-ms for improvement 4 depends on this. Open.

LFO object (class used per voice at voice+0x560 and +0x57c, and globally at engine+0x4030 and +0x404c), tick fcn.42016648:
- +0 held sample (int16), +2 phase (uint16), +4 previous phase, +6 retrigger flag, +8 rate (float), +0xc rate modifier, +0x10 skew/breakpoint (float, warps the phase; where it is set not found), +0x14 shape (byte), +0x18 output (float, -1..1).
- Phase step per tick = min(8191, int(rate*127*64 + ...)) of a 65536 phase, so rate 1.0 is about 0.124 cycles per tick.
- Shapes (value 0..6): 0 sine (1024-entry ROM table at 0x3c071160, interpolated), 1 triangle, 2 square, 3 falling saw, 4 rising saw, 5 white noise (new random every tick), 6 sample and hold (new random at phase wrap or zero crossing). Firmware strings include "Saw", "Saw Up", "S & H", "Sine", "Triangle", "Square"; the exact name per index is assumed from this order, index 5 name not found.

LFO parameters (per part k): LFO1 Speed 0x26 (+0x98), Ratio 0x27 (+0x9c), Clocked 0x28 (+0xa0), Shape 0x29 (+0xa4), Sync 0x2a (+0xa8); LFO2 the same at 0x2b..0x2f (+0xac, +0xb0, +0xb4, +0xb8, +0xbc).
- Free rate: rate = clamp(Speed + modmatrix LFO speed offset (voice+0x774 for LFO1, +0x77c for LFO2), 0..1) squared.
- Clocked rate: rate = tempo value (clock object at engine+0x4068, as float) x ratio table[int(Ratio+0.05), max 18] (19 floats at RAM 0x3fcbb478, built at startup, values not yet dumped).
- Sync (value 0 vs >0): 0 = own LFO per voice (voice+0x560 / +0x57c, updated in the voice). >0 = shared global LFO (engine+0x4030 / +0x404c), updated once per tick by fcn.4037e71c only when Sync > 0. Shared LFOs are therefore one phase for all voices; sync value above 0 is not used for anything else seen.
- LFO output pointers: mod source 4 (LFO1) and 7 (LFO2) are re-pointed at set time to the chosen LFO object output (+0x18).
- Derived sources recomputed each tick: LFO1*MW (voice+0x598) = ModWheel x LFO1; LFO1*AT (+0x5a0) = (ChannelPressure + PolyAT) x LFO1; LFO2*MW (+0x59c); LFO2*AT (+0x5a4).

Mod matrix (object at voice+0x5e0, called "M"):
- 8 rows, stride 8 bytes, at voice+0x7e4: byte source id, byte destination id, float amount (0..1 param; used as amt*2-1 so 0.5 = none). Row n: source param k=0x3a+3n (value at +0xe8+0xc*n), dest +1, amount +2.
- Source id 0 = off, ids 1..34 index the pointer table M+0x14+8*id. Destination id 1..26 (0 = off, above 26 ignored).
- Per row: accumulator[dest] += source value x (amt*2-1); destinations flagged as curved (dest ids 3, 4, 5) use x*|x| instead. Each accumulator is clamped to -100..+100. The loop runs once per tick over all 8 rows after clearing the accumulators it touches (stamp counter at M+0x244).
- Accumulator for dest id at voice+0x70c+8*id.

Source ids (names from the UI list order, ids from constructor; ids 1..28 and 30..34 consistent, id 29 unnamed):
1 Wav Env (voice+0x4f8), 2 Filt Env (+0x538), 3 Amp Env (+0x518), 4 LFO1, 5 LFO1*MW, 6 LFO1*AT, 7 LFO2, 8 LFO2*MW, 9 LFO2*AT, 10 Mod Wheel, 11 Pitch Bend, 12 Note PB, 13 Ch Press, 14 Poly AT, 15 MPE Y, 16 Pedal, 17 Expression, 18 Breath, 19-28 CC 22..31 (part values +0x58..+0x7c), 29 voice+0x18 (name not found, per-note glide step), 30 Velocity (voice+0x10), 31 Rel Velo (voice+0x14), 32 Random (new random 0..1 per voice start, M+0xc), 33 Rand Bipol (-1..1, M+0x10), 34 Constant (1.0, M+8).
Source to MIDI value pointers for ids 10-18 come from the part controller block (voice+0x5cc, +0x5d0, +0x5d4, +0x5d8); mapping of each pointer to its MIDI source was inferred from the UI name order, not traced to the MIDI parser.

Destination ids (accumulator offset, how it is used):
1 Wave 1 position (+0x714, x63 in position), 2 Wave 2 position (+0x71c), 3 pitch of both oscillators (+0x724, x24 semitones), 4 Osc1 pitch (+0x72c), 5 Osc2 pitch (+0x734), 6 Osc1 volume (+0x73c, /5120 scale), 7 Osc2 volume (+0x744, /5120), 8 Dirt/noise amount (+0x74c), 9 Cutoff (+0x754), 10 Resonance (+0x75c), 11 Volume (+0x764), 12 level/pan stage (+0x76c, used with pan 0x4e0 and +0x4f4), 13 LFO1 speed (+0x774), 14 LFO2 speed (+0x77c), 15-18 Wave Env A D S R (+0x784..+0x79c), 19-22 Filter Env A D S R (+0x7a4..), 23-26 Amp Env A D S R (+0x7bc..+0x7dc).
UI destination names found in firmware: Wave 1, Wave 2, O1 Pch, O2 Pch, O1 Vol, O2 Vol, Dirt, Cutoff, Resonance, Vol, LFO 1 Speed, LFO 2 Speed, Wave Env Att/Dec/Sus/Rel, Filt Env Att/Dec/Sus/Rel, Amp Env Att/Dec/Sus/Rel. Names for ids 3 (pitch of both) and 12 were not found, so the UI index to id mapping between Wave 2 and Dirt, and after Vol, is not confirmed.
Wave position range: Wave 1/2 offset is added as mod x 63, so a full-amount modulation sweeps the whole table.
Envelope time destinations add to the envelope parameter (0..1) before the table lookup and are clamped 0..1.

Open points:
- Tick rate (see correction), ratio table values, clock tempo units.
- LFO skew source (+0x10) and the retrigger flag (+6) setter.
- UI index to destination id mapping for ids 3 and 12.
- Whether the amount curve flag list is only ids 3-5 (the dump showed ids up to 0x18 only).

## Audio loop and timing (round 10)

Two FreeRTOS tasks form a double-buffered pipeline, one block = 16 samples. Sample rate: engine is initialised with 44100.0 (0x472c4400 passed to fcn.4200d77c in the engine task), so 44.1 kHz is confirmed for the engine. Whether the I2S hardware is clocked at the same rate was not decoded (I2S config template at 0x3c073968 not read).

Engine task `fcn.4200ccc8` (producer), one loop per 16-sample block:
1. Wait for a notification (fcn.40386d34), record start time (fcn.4037a37c).
2. Drain the engine event queue and dispatch each 24-byte event with `fcn.4200d7f4` (see round 5).
3. Render: `fcn.4200d7bc` -> note processor render `fcn.42013e30`, which does: arp/seq clock tick `fcn.42017670(obj+0x728)`, `Time__update` (microsecond clock), engine mixdown (engine vtable +0x60 = `fcn.42015204`), then `fcn.42016e08`. So the arp/sequencer is stepped once per 16-sample block (0.363 ms resolution).
4. Mixdown `fcn.42015204`: for each of the 4 parts whose active byte (part+0xcb0) is set, call part render `fcn.4037a92c`, then add its 16 L + 16 R samples to one of 3 stereo buses chosen by the per-part parameter k=0x38 "Destination" (engine array offset 0x3620 + 0x14c*part): value 1 -> bus 1, value 2 -> bus 2, anything else -> bus 0. Buses are 6 buffers of 16 floats at 0x3fca37b0, 37f0, 3830, 3870, 38b0, 38f0 (L/R pairs). Firmware strings "FX 1", "FX 2", "Out" are presumably the Destination names; which value is which is unconfirmed.
5. Part render `fcn.4037a92c` (two variants for mono/stereo): calls the part's tick function once (vtable +0x10 of the layer DSP object, `fcn.4037e71c`), then loops over the 8 voice slots (stride 0x70) calling `fcn.4037a794`, which calls the voice render `fcn.4037bfb0`. Voice outputs are summed into the part's 16 L + 16 R buffers.
6. Signal the output task (semaphore at *0x3fca3964), measure elapsed microseconds into 0x3fca37a0 (probably a CPU load figure), loop.

Output task `0x4200ca8c` (labelled CircularBuffer__drop_1 in the decompile; really the audio output and input task), one loop = 64 frames:
1. 4 x `fcn.4200d09c` (one per 16-sample block): update clock, wait for the engine's semaphore (*0x3fca3964), copy the 6 bus buffers, release the engine task (semaphore *0x3fca3968), then run the effects on the buses (this is the effects code, roadmap item 4) and write 16 stereo frames.
2. One I2S write `fcn.42026b28` of 0x200 bytes = 64 frames (8 bytes per frame, so 32-bit stereo slots, assumed) = 1.451 ms of audio.
3. Poll the MIDI/input queue (0x3fcd2210, up to 0x20 items) and feed each to the MIDI parser `fcn.42019bdc` / `fcn.42019830`. So incoming MIDI is processed once per 64 frames.
I2S setup `fcn.42026ba0`: pins MCLK 7, BCLK 9, WS 10, DOUT 8 (order assumed from the config struct), MCLK multiple 256.

Control tick (answers the round 9 open point):
- `fcn.4037e71c` is called once per part per 16-sample block. Counter +0x4024 += 16 per call, threshold +0x4028 += 49 per tick. So a tick is 49 samples (about 1.11 ms, about 900 Hz), with random +-32 jitter on the threshold when the part setting at +0x50 is above 0.25.
- Voices run LFO products every 4th tick and envelope stage updates when (tick count & 7) == 0, i.e. every 8 ticks = 392 samples = 8.89 ms.
- Implied envelope times, with c[i] = ROM[i]/16/32767 per envelope update and 99% time = 4.6 x (step / c). Three candidate update intervals, because the result at 8.89 ms looks implausibly slow (index 127 would be over an hour):
| knob index | c | 99% at 8.89 ms | 99% at 1.11 ms (every tick) | 99% at 0.363 ms (every block) |
|---|---|---|---|---|
| 2 | 0.0582 | 0.70 s | 0.088 s | 0.029 s |
| 40 | 0.0042 | 9.8 s | 1.2 s | 0.40 s |
| 63 | 0.00085 | 48 s | 6.0 s | 2.0 s |
| 90 | 0.00013 | 316 s | 39 s | 12.9 s |
| 127 | 0.0000095 | 4292 s | 536 s | 175 s |
  The correct interval is NOT confirmed. It needs a hardware measurement (attack time at a known knob position) or a closer look at the gate at voice+0x5bc. The ms display feature (improvement 4) must wait for that.

Voice allocation (part note-on `fcn.4037a4b0`, part object: 8 voice slots at +0x10 stride 0x70, last-started slot at +0x390, polyphony limit at +0x392, mono/legato flag at +0x397, note state table at +0x62c):
- Velocity 0 is treated as note-off. Note state table set to 1 and the newest note and its velocity are stored (+0x394, +0x396).
- Mono/legato flag set: the same voice (slot +0x390) is re-pitched (glide) instead of a new voice, with a variant depending on the voice state.
- Otherwise round-robin: candidate slot = (last started + 1) mod 8. Voice slot state byte at +0x11: 0 free, 1 and 2 busy, 3 being stolen. If the candidate is free and the count of active voices is below the polyphony limit, the note starts there. If not, a voice is stolen: the chosen slot is set to state 3, a fast fade-out is started through the voice's vtable +0x1c, and the new note is stored as pending in the slot (+0x30 note, +0x34 velocity) and starts when the fade ends. Which busy voice is chosen is decided by the three candidate checks (state 2, state 1 or the +0x12 flag); the exact priority is not fully traced.

Open points:
- Which update interval the envelopes really run at (see table).
- I2S slot width and hardware sample rate.
- Meaning of Destination values and the exact steal priority.

## Effects (round 11)

Where it runs: the output task (see round 10) calls `fcn.4200d09c` once per 16-sample block. That routine waits for the engine's buffer, calls the FX processor `fcn.4200e700(fx, bus0L, bus0R, bus1L, bus1R, bus2L, bus2R)` with fx = task struct + 0x188, then converts the result to 32-bit signed integers (float x 2^31, clamped) for I2S, left and right. So the audio output is 32-bit stereo (confirmed).

Signal chain inside `fcn.4200e700` (read from the code, one pass per block):
1. Bus 0 -> FX1 (object pointer at fx+0x1488, process = vtable +0x10) -> temp.
2. Add bus 1 to the FX1 output.
3. Sum -> FX2 (pointer at fx+0x148c) -> output.
4. Add bus 2 to the FX2 output.
5. Master stage: a fixed 2nd-order IIR filter on left and right (coefficients fx+0x60..0x80, look like a DC block or high-pass, role not confirmed), then a peak limiter (peak detect over groups of 4 samples, attack coefficient fx+8, release fx+0xc, smoothed gain applied to both channels), then a final per-channel function `fcn.4200d938` (not read).
So the per-part Destination setting (round 10) is: bus 0 = FX 1 (goes through FX1 then FX2), bus 1 = FX 2 (skips FX1), bus 2 = Out (skips both). The firmware strings "FX 1", "FX 2", "Out" fit this order.

Effect types (FX1Type and FX2Type, 10 values each; same list for both slots). Object per type, base offset from the FX base (these offsets were checked against where each effect's parameters are stored):
| value | effect | object offset | constructor |
|---|---|---|---|
| 0 | Bypass | +0x94 | (dummy) |
| 1 | Chorus | +0xa0 | fcn.42010938 |
| 2 | Flanger | +0x98c | fcn.42012978 |
| 3 | Phaser | +0xa40 | fcn.420132bc |
| 4 | Tremolo | +0x13fc | fcn.42013cac |
| 5 | Drive | +0x7f0 | fcn.420118d8 |
| 6 | Compressor | +0x2a4 | fcn.4200fa2c |
| 7 | EQ | +0x83c | fcn.42011c64 |
| 8 | Delay | +0x6e4 | fcn.420113e0 |
| 9 | Reverb | +0xfd0 | fcn.42013710 |
The UI strings for the type list are "Off"... "Chorus, Flanger, Phaser, Tremolo, Drive, Compr, EQ, Reverb"; no "Delay" string was found, but the parameter group and object at +0x6e4 are clearly a delay. The type value is clamped (values above 9 ignored). Both slots can hold the same effect type, but each object exists only once, so FX1 and FX2 set to the same type would share one object (not verified, risk when planning changes).

Parameter table: 75 FX parameters in a descriptor table at 0x3c06f5c4 (same 32-byte entry format as the sound table). FX parameter id = position in this table starting at 0; UI parameter index = 0x150 + id. A UI parameter change goes through `Engine__sendEngineEvent` (fcn.4200d52c): for event type 6 it converts the id with `fcn.42018aec`; if it is an FX id it calls the FX setter `fcn.4200de3c(fx, id, value)` immediately in the sender's context (not through the engine queue). Defaults in brackets, discrete step counts as "n steps":
- 0 FX1Type, 1 FX2Type (10 steps, default 0), 2 BypassDummy
- Chorus 3-12: DryWet [0.75], Depth [0.4], Speed [0.2], Feedback [0.5], Center [0.06], Spread [0.75], Shape (2 steps, [1]), Stages (4 steps, [1]), HighCut [0], LowCut [0]
- Flanger 13-19: DryWet [0.5], Depth [0.65], Speed [0.3], Feedback [0.6], Center [0.1], PhaseDiff [0.5], Shape (2 steps, [0])
- Phaser 20-29: DryWet [0.5], Depth [0.3], Speed [0.5], Feedback [0.6], Center [0.5], Spread [0.7], PhaseDiff [0.5], Shape (2 steps, [0]), Stages (5 steps, [2]), Model (2 steps, [0])
- Tremolo 30-34: Depth [0.75], Speed [0.4], PhaseDiff [0.5], Shape (3 steps, [0]), Smooth [0.75]
- Drive 35-38: DryWet [0.35], Amount [0.7], Gain [0.35], Type (5 steps: PNP, Tube, PickUp, Diode, Crunch)
- Compressor 39-46: Ratio [0.2], Threshold [0.5], InGain [0.5], OutGain [0.5], AutoGain (2 steps, [1]), Attack [0], Release [0.2], LookAhead [0.5]
- EQ 47-58: DryWet [1.0], Gain [0.5], LowFreq [0.1], LowGain [0.5], MidFreq [0.3], MidGain [0.5], MidQ [0.7], Mid2Freq [0.6], Mid2Gain [0.5], Mid2Q [0.7], HighFreq [0.9], HighGain [0.5]
- Delay 59-68: DryWet [0.5], Left [0.3], Right [0.35], Link (2 steps, [1]), Synced (2 steps, [0]), RatioL (16 steps, [6]), RatioR (16 steps, [6]), Feedback [0.75], HighCut [0], LowCut [0]
- Reverb 69-74: DryWet [0.25], Time [0.75], Color [0.5], ModDepth [0.2], ModRate [0.5], PreDelay [0.15]
Entries with a display-format pointer (Drive Gain, Compressor Ratio/Threshold/InGain/OutGain/Attack/Release/LookAhead, Delay Left/Right) use the dB / ratio / ms / s formatters seen in the strings ("%+.1f dB", "1 : %.1f", "%.0f ms", "%.1f ms", "%.3f ms", "%.3f s").

Where each parameter lands (offset from the FX base; ids not listed were not resolved; "ramp" = value is smoothed with a target/step/count structure, so changes glide):
- Chorus (obj +0xa0): 3 DryWet 0xac, 4 Depth ramp at 0xc4, 5 Speed 0xb0, 6 Feedback 0xb8, 8 Spread 0xb4, 9 Shape 0x264 (int), 10 Stages 0x174 (int), 11 HighCut 0xbc, 12 LowCut 0xc0. Id 7 (Center) not resolved.
- Flanger (obj +0x98c): ids 13-18 at 0x998, 0x99c, 0x9a0, 0x9a4, 0x9a8, 0x9ac; 19 Shape 0xa30 (int).
- Phaser (obj +0xa40): 20-26 at 0xa4c, 0xa50, 0xa54, 0xa58, 0xa5c, 0xa60, 0xa64; 27 Shape 0xfbc (int), 28 Stages 0xf80 (int), 29 Model 0xa7c (int).
- Tremolo (obj +0x13fc): 30 Depth 0x145c, 31 Speed ramp at 0x1450, 32 PhaseDiff 0x1460, 33 Shape 0x1448 (int), 34 Smooth 0x1468.
- Drive (obj +0x7f0): 35 DryWet 0x7fc, 36 Amount 0x800, 37 Gain 0x804, 38 Type 0x838 (int, 0..4; same five waveshapers as the voice drive, see round 8).
- Compressor (obj +0x2a4): 39 Ratio ramp at 0x2e0, 40 Threshold ramp at 0x2d0, 41 InGain ramp 0x2b4, 42 OutGain ramp 0x2c4, 43 AutoGain 0x324 (bool), 44 Attack -> coefficient at 0x300, 45 Release -> coefficient at 0x304, 46 LookAhead 0x61c (lookahead delay line, read index 0x614). Attack coefficient = fcn.4203d85c(0.1, 1 / (value x 0.2 x 11025)) and release uses (value x 2 x 11025), i.e. times in samples, apparently up to 0.05 s attack and 0.5 s release at 44.1 kHz (the exact meaning of the 0.1 base is not confirmed).
- EQ (obj +0x83c): 47 DryWet 0x928, 48 Gain 0x92c, 49-58 at 0x938 (LowFreq), 0x93c, 0x944, 0x948, 0x94c, 0x954, 0x958, 0x95c, 0x964, 0x968; the flag at 0x978 is set on every EQ change (coefficients are recalculated on the next block).
- Delay (obj +0x6e4): 60 Left ramp at 0x6f0, 61 Right ramp at 0x700, 62 Link 0x728, 63 Synced 0x72c (bool), 64 RatioL 0x730 (int), 65 RatioR 0x734 (int), 66 Feedback 0x714, 67 HighCut 0x71c, 68 LowCut 0x720. DryWet id 59 not resolved.
- Reverb (obj +0xfd0): 69 DryWet 0x13d4, 70 Time 0x13d0, 71 Color 0x13d8, 72 ModDepth 0x13e0, 73 ModRate 0x13e4, 74 PreDelay 0x13dc; flag 0x13f8 set on change.

Other things found in the FX/IO task struct (task struct + offsets):
- +0x1618..0x1630: an LFO that mirrors the LFO speed of the selected part (event types 11 and 12 choose the part and LFO number, speed squared is stored per part at +0x1634/+0x1638); its on/off state is written to the global at 0x3fcd1dd4 (almost certainly the front panel LFO LED).
- +0x1680..0x1684: a second tick counter of the same 16/49 kind, used for the LED LFO.
- Event type 10 sets a 0..511 value (divided by 511) at +0x168c with a smoothing ramp into +0x1688. Nothing was found that reads +0x1688 yet; its role (master volume, Flavour or other) is open.
- +0x1698 flag enables CPU timing: elapsed microseconds of the whole routine are smoothed (95/5) into +0 and the peak kept at +4 (a load meter).

Open points:
- Chorus Center, Delay DryWet, and the remaining unresolved ids above.
- What the last master filter and `fcn.4200d938` do exactly.
- The actual DSP (delay line lengths, reverb topology, phaser all-pass count, limiter threshold) is not documented; each effect class would need its own pass if you want to change an effect.
- Whether FX1 and FX2 can safely hold the same effect type.
- Role of the 0..511 event type 10 value.

## Settings and menus (round 12)

Settings table: 0x3c0752a0, 134 entries x 24 bytes {section str ptr, label str ptr, min, max, default, visible byte}.
Values live in the Globals object at 0x3fccfd14 (int32 values[134]; same object as controller+0x18cc). Getter fcn.4204f8d0(obj, i), setter fcn.42029120 (marks dirty).
Persistence: loaded at boot by the controller init (fcn.420193ec) via Global__load fcn.42029170; the MIDI/control task fcn.420191ec calls autosave fcn.420291e4 every 200 loop passes. Tolerates smaller stored blobs.
First boot: NVS namespace "Factory" key "Data" missing -> flag written, factory presets initialised ("Presets ...").

Visible items (13), what they do, and display strings (formatter fcn.4202925c):
- 0 MIDI Settings / Omni Mode 0..1 def 1 (Off/On). On = accept all channels.
- 1 MIDI Settings / Receive Chan 0..15 def 0 (shown +1). Used only when Omni is Off: message dropped unless (channel - Receive Chan) is below 1 (below 4 when sound params 0x229/0x22a equal 4 and 3, probably a multi-channel mode; unconfirmed).
- 2 MIDI Settings / Send Chan 0..15 def 0 (shown +1). Outgoing short messages get their channel nibble replaced with this value (fcn at 0x42019xxx, status < 0xf0 only).
- 3 MIDI Settings / Device ID 0..126 def 0. Compared with byte 7 of incoming sysex; 0x7f = broadcast. Also written into outgoing sysex.
- 4 Pitch Settings / Transpose -12..12 def 0 ("Off" at 0, else "%+d"). Sent to engine as event type 7.
- 5 Pitch Settings / Master -100..100 def 0 ("0 c" or "%+d c"). Sent to engine as event type 8.
- 6 Display Settings / Brightness 0..19 def 17 (shown +1). Passed to display object 0x3fcd1de4 (fcn.42027360).
- 7 LED Settings / Brightness 0..16 def 14 (shown +1). Value squared, clamped to 255, passed to LED driver 0x3fcd1b78 (fcn.42026b4c).
- 8 Pot Settings / Mode 0..1 def 1: 0 "Absolute", 1 "Relative". Mode 1 enables the pickup/blend logic in the knob scan (fcn.4201ea04); 0 reads the knob directly (the second branch was only skimmed).
- 10 Sync Settings / Source 0..1 def 0: 0 "Intern", 1 "Extern". Stored as a flag at controller+0 (value != 0).
- 131 MPE Settings / Status 0..1 def 0 ("Deactive"/"Active"). Sent to engine as event type 9. When on, incoming channel messages go through the MPE parser fcn.42028f70 instead of the Omni/Receive Chan filter.
- 132 MPE Settings / Max Channel 1..15 def 15 (displayed +1, so 16; reason unknown). Copied to controller+0xb80 for the MPE parser.
- 133 MIDI Settings / SysEx Out 0..1 def 0: 1 = "DIN", else "USB". Chooses the port for sound dumps (MidiSysexController__dumpSound).
Hidden: 9 Last Loaded Preset (restored at boot), 11..130 = CC map entries for CC 0..119.

Apply-all: controller event 0x20 pushes items 6, 7, 4, 5, 131 and the sync flag (10); it is the path used after load and after a settings change.

CC map: item 11+cc holds the parameter index (-1 = none, range up to 560). RAM copies: cc->param u16 table at controller+0xc6c (0xffff = none), param->cc u8 table at controller+0xd5c (0xff = none). Set by fcn.420197c0(cc<120, param<0x231), cleared by fcn.420197f4(cc). CC handler fcn.420199d0: CC 1 also writes extra param 0x230 (value/127); CC 32 stored at controller+0x1af6; stepped params scale by (count-1); bipolar params centre at 64 ((v-64)/126+0.5); CC 124 handled specially (unconfirmed).

Controller event handler (fcn.4201a0d4, 1727 bytes) event ids seen: 0 sysex state, 6 set parameter (value x 1e-6), 8/9/0xa/0xb/0xf sound library edits (init, load, replace, remove, copy), 0x13 dump sound over sysex, 0x14 dump range, 0x16 push all params to engine, 0x18/0x19 batch operation over the library, 0x1a "Init Presets", 0x20 apply-all, 0x22 and 0x23 engine control. Meanings of 8/9/0xa/0xb/0xf are by call pattern, not tested.

Not mapped yet: Sound list Actions menu, Save page, CC Mappings page drawing, Info popup text, string table of the settings page layout.

## MIDI details (round 13)

Controller singleton: 0x3fcce448 (MIDI-in entry fcn.42019bdc; Globals sit at 0x3fccfd14, engine event sink 0x3fca3974).

Input routing (fcn.42019bdc):
1. The UI sees every message first (UserInterface__processMidiInput; the Map CC page uses this, see below).
2. Status >= 0xf0 goes to the system/clock handler fcn.4201beac (no channel filter).
3. Channel filter: MPE on (setting 131) -> MPE parser; otherwise Omni/Receive Chan rule from round 12.
4. 0xB0 -> CC handler fcn.420199d0 (CC map, round 12). Other messages become 24-byte engine events {type, a, b, c, d, e} via Engine__sendEngineEvent:
   type 0 note on (note, velocity, channel), 1 note off (also 0x90 with velocity 0), 2 pitch bend (14-bit, lsb | msb<<7), 3 controller (id, value), 4 channel pressure, 5 poly aftertouch.
5. Program Change (0xC0): preset = bankLSB(CC 32, stored at controller+0x1af6) * 128 + program; accepted if < 360 and the slot is used (fcn.4204f900); stored as pending load at controller+0x1af4 (applied elsewhere). CC 0 bank MSB is not used in this path.

Clock sync (fcn.4201beac, tempo fcn.4201c074; state in the controller object, offsets from 0x3fcce448; timestamps are 64-bit microseconds):
- 0xF8 clock: stored in a 64-entry ring of timestamps (+0x8, stride 8, head +0x208), tick counter +0x228, clock count since start +0x21c. Only while byte +0x220 is non-zero (what sets it: unconfirmed).
- 0xFA start: state +0x218 = 2, count reset. 0xFB continue: state 3. 0xFC stop: state 5. 0xFF reset: state 5, position and ticks 0. State 4 = running (first clock after start/continue). 0xF2 song position: +0x224 = msb<<7|lsb (in 16ths), tick counter = SPP x 6.
- Tempo: BPM = 2,500,000 / (interval in microseconds between clocks), i.e. 24 PPQN. Accepted range 5 to 1000 BPM; outside it, or when the last clock is older than 0.5 s, the last good value is kept (default 120). Position output: beats = clock count / 24, and SPP beats = SPP x 0.25.
- fcn.4201c074 fills a transport record (tempo double at +8, beat position at +0x18, SPP at +0x28, valid flags) that the engine clock object consumes (callers at 0x4200xxxx region); with Sync Source = Intern (*controller == 0) it returns tempo 120 with flags 0. Exact consumer of the record not traced.

MPE parser (fcn.42028f70, object at controller+0xb40):
- Zone sizes at +0x40 (lower zone, manager channel 1) and +0x44 (upper zone, manager channel 16). Setting 132 (Max Channel) is copied into the lower zone size (+0xb80 = +0x40). Both zones can also be set by MIDI: RPN 6 (CC 101 = 0, CC 100 = 6, CC 6 data 0..15) on channel 1 (lower) or 16 (upper); data 0 turns that zone off.
- Per-channel arrays (4 bytes each): +0x00 note, +0x5c pitch bend (default 0x2000), +0x9c channel pressure, +0xdc CC 74 (slide, "MPE Y" mod source).
- Note on stores the note and copies the channel's current bend, pressure and CC 74 to +0x120/+0x124/+0x128 with flag +0x11c = 1. The router then sends them as events 2, 4, 3 (id 74) before the note on, so a note starts with its initial expression. Note off resets that channel's values.

MIDI learn / Map CC (UI hook fcn.4201dee4): when the Map CC page is active, any incoming CC (status 0xB0, number < 120, channel not checked) is mapped to the parameter selected in the UI (UI+0x1290) through fcn.420197c0, then a 3 s popup shows the CC number and parameter name. Mapping removal from the Mappings page: fcn.4201dab0 -> fcn.420197f4. No other "learn" code found.

Sysex (MidiSysexController__processMessage fcn.4201bbf8). Header as read from the decompile (offsets in the message buffer, byte positions approximate): 0x3E (Waldorf), 0x2F or 0x00, device ID (must equal setting 3 or be 0x7F), command. Commands:
- 0x10 receive sound dump (onSoundDump), 0x11 request current sound dump (dumpSound of the sound at 0x3fccf428)
- 0x12 request preset N dump (N 14-bit, < 360 = 0x168), 0x13 receive preset dump
- 0x14 select/check preset N (14-bit): replies status 0 if the slot is used, 1 if not (exact action unconfirmed)
- 0x20 request preset names (start, count, up to 360)
- 0x30 screen dump (dumpScreen), 0x40 firmware version
- Reply status codes seen in sendResponse: 0 ok, 1 failed/empty slot, 2 bad index, 6 bad range. Output port from setting 133 (DIN or USB). Preset slots: 360 (0x168).

Startup arpeggio player (fcn.4201c280/fcn.4201c358, state byte controller+0xfdd 1 -> 2, starts about 4 s after boot): feeds note on/off, velocity 100, into the MIDI-in path using a 2-entry table at 0x3fca04c4 (stride 12; first entry base note 60, step 3 semitones, 4 notes = 60, 63, 66, 69). Purpose unconfirmed (test or demo pattern). It is a candidate to disable if it ever sounds at boot.

Not done: USB MIDI class driver details, DIN UART setup and baud, active sensing, system exclusive size limits, what sets clock-enable byte +0x220.

## Storage (round 14)

Backend: ESP-IDF NVS on a partition labelled "data" (not the default "nvs"). Storage object at 0x3fcd2e24; wrappers Storage__load fcn.420278a0, Storage__save fcn.42027938 (commit after write), Storage__erase fcn.420279e0, Storage__loadGlobalsPartial fcn.42027b54. Load fills the buffer with 0xFF and returns 0 when the key is missing or the stored length differs (except globals, which accept a shorter blob).
Init fcn.4202780c: nvs init of "data"; on error 0x110d (no free pages) or 0x1110 (new version) the partition is erased and initialised again. Partition size and offset are NOT in the UF2 or ELF (partition table lives outside the app slot): unknown.

Namespaces and keys actually used (no others found in the code):
- "Factory" / "Data": 1-byte blob, first-boot flag. Missing at boot -> flag written, factory presets re-created (see below).
- "globals" / "default": blob of 0x218 bytes = 134 x int32 settings values (round 12). Written by the debounced autosave.
- "sounds" / "sound-%03i" (slot 0..359, key built with snprintf into a 17 byte buffer): one blob per preset, size 0x11a8 = 4520 bytes. Erase on slot remove deletes the key.
Not stored in NVS: wavetable cache, event/log data, MIDI CC map separately (the CC map is part of globals items 11..130).

Sound blob format (4520 bytes):
- 32-byte header: bytes 0..23 name (NUL terminated, first byte masked with 0x7F when sent by sysex), byte 0x18 sent in the preset-names sysex after the name (likely category), byte 0x19 = format version (8 = current, see Round 36).
- Then 0x231 (561) records of 8 bytes: {4-byte parameter tag, float32 value}. The tag is built by fcn.42018b3c: for params 0..0x14f (4 layers of 84) three chars from the per-parameter record in the UI table plus the layer digit ('1'+layer) in the low byte; for params 0x150..0x230 four chars from the global parameter table. So the file is keyed by name tag, not by position, which lets a later firmware reorder parameters.
- Save path: snapshot buffer at 0x3fcd3420 (header copy then the tag/value pairs), flag and target slot in the Sound object (+0x8e4 dirty, +0x8e8 slot); fcn.42029dc0 in the control task writes it when the library mutex frees (fcn.42029d84 -> fcn.42027aac -> NVS), then copies the 32-byte header to the RAM index.
- Load: sound loads are done via fcn.42027a7c (key by slot) in three places (sysex request paths and UI load); the tag/value pairs are matched back to parameters (matching code not traced).

Library RAM index (fcn.42029c84 at boot, object 0x3fcd32b4): 360 slots (0x168). Allocates 0x2d00 bytes = 360 x 32-byte headers, plus a used flag per slot at object+4+4*slot. Boot scans all 360 keys by reading each blob once (fcn.42027adc gets the stored length, 0 = free), so boot time grows with the number of used slots.

Factory presets: fcn.42029e50 parses a stream of F0..F7 sysex messages, feeds each to the sysex handler and saves it to the next slot. 12 sysex messages with header F0 3E 2F 00 10 are embedded in flash rodata at 0x3c076130 to 0x3c0789f0 plus ~0.5 KB (sizes 1016, 4 x 990, 480, then the same set again; both sets total 5456 bytes, roughly one 4520-byte sound packed to 7 bit). So likely two embedded sounds (init material), not 360 factory presets; the 360 factory presets must come from elsewhere (the "Presets ..." init path) or be absent. Unconfirmed.

Sysex header confirmed from the embedded data: F0, 3E (Waldorf), 2F (Protein), device ID, command byte, then payload. Command 0x10 = sound dump.

Consequences for the improvement list:
- Item 3 (custom wavetables): there is no existing user storage for wavetables, so tables must be part of the firmware image as planned.
- Any new persistent setting (item 1 latch mode toggle, item 4 time display mode) fits into the globals blob only if the settings table grows: loader accepts a shorter stored blob, and new entries take their table defaults, so adding settings at the END of the table is safe for existing units. Changing the order or size of existing entries is not.
- Sound blobs are keyed by 4-char tags, so adding parameters (for example envelope selector) is backward compatible if tags are new and the record count (0x231) and buffer size (0x11a8) are raised together; sizes are checked on load, so old presets with the old length would fail to load unless the check is relaxed. Needs care.

## Flavour dial (round 15)

What it is: global parameter UI 0x22f, tag "flav", engine id 0x14, name "Flavour", continuous (steps 0), default 0.17, stored in every preset as a global record. It is edited with the second knob of the Master section (knob id 14, handler fcn.42021420): the first knob of that section (id 13) sends the master volume as engine event type 10 (raw 0..511 pot value); the Flavour knob reads the pot through the same scan routine as the others (fcn.4201ea04, honours Pot Mode), writes Sound value 0x22f, calls fcn.42019024(0x22f) to push it to the engine, and shows "Flavour" on the display (the display falls back to "Volume" after 3 volume updates, flag controller UI +0x28).

Path in the engine: param event type 6 -> note processor setter (fcn.42013f40): id 0x14 has no case there -> forwarded to the engine virtual setParam (vtable 0x3c070218 +0x40 = fcn.420157e0). setParam only stores the float at engine+0x32ec+4*id (Flavour = engine+0x333c) and returns; the only id with extra behaviour below 0x95 is 0xe (LayerMode).

Consumers: none found. No code reads engine+0x333c, the note processor ignores id 0x14, and no mod destination, FX or voice path references it. The generic getter (engine vtable +0x3c, fcn.42014cd0, reads engine+0x32ec+4*id for id < 0x1e1) exists, but no caller that reads id 0x14 was found. Conclusion (not 100 percent proven because virtual calls are hard to trace): in v1.02 the Flavour dial is stored, saved, displayed and sent to the engine but has no audible effect. It sits in the chord/scale block (ChordMode 0x10, ChordType 0x11, ChordScaledType 0x12, ScaleType 0x13, Flavour 0x14), so it was probably meant as a chord voicing/variation amount that is not implemented in this version.

Related findings made while tracing:
- Event type 10 (master volume) value is stored at engine+0x168c and ramped into +0x1688; the only reads of +0x1688 found are the ramp itself. Where the ramped value is applied to the output was not found (probably in the mixdown or the master filter path); still open, same as in round 11.
- Engine id 0xe LayerMode (UI 0x229): value 0..3 = single layer (only that part active, polyphony 8), 4 = Multi: parts with ActiveInMulti > 0 (last per-part parameter, index 107, 2 steps) are active and share the 8 voices evenly (8 / number of active parts). Done in fcn.420156d8 via part polyphony limit fcn.4204e824 (part+0x392) and part active byte (part+0xcb0 stride 0xcb8). LayerAlloc (id 0xf) is stored but its use was not traced.

Idea for later: Flavour is a free, saved, continuous global with a physical knob, so it could be repurposed (for example as the envelope time scaling or arp latch amount) with little UI work: only the consumer code would be new. Not decided.

## Boot, USB and system (round 16)

Entry: ELF entry 0x40379204 (ESP-IDF startup). The application's own start is the audio/output task fcn.4200ca8c, which first runs fcn.4200ca58:
1. fcn.42026fd0 hardware and library init (order below). Arguments: LED/button object 0x3fcd1b78, 64, 44100 (0xac44).
2. Factory test mode check: if two flags in the button object (0x3fcd1b78 + 0x11c and + 0x12c) are both 1, fcn.42026910 runs and never returns. Which two buttons these are is not identified.
3. Engine init fcn.4200ce38 (engine object 0x3fca3974, sample rate 44100.0f). It logs free memory, creates the engine tasks and semaphores; an "Extra Engine task" exists (second engine task, error text reports internal RAM free if creation fails), so the engine uses two tasks (core and priority values not read).
4. Controller init fcn.420193ec (controller 0x3fcce448): factory flag, settings load, CC map tables, sound library scan (round 14), creates the "Helper task", then the loop continues into the I2S output loop (round 10).

Hardware init order in fcn.42026fd0: fcn.42028904, fcn.42026f9c, fcn.420283a4, fcn.420274f0 (display/input object +0x26c, 796 bytes), fcn.42028884, boot splash fcn.420192cc if the button test passes, USB fcn.42028050, I2S fcn.42026ba0, MIDI UART fcn.42027cec, NVS init fcn.4202780c, per-LED-channel init loop, GPIO 6 and GPIO 21 configured as outputs and driven low (purpose unknown), then fcn.420280b0 (reads MIDI input stream bytes into the MIDI parser buffers).

Boot splash (fcn.420192cc): sets LED brightness 200, draws text with the font tables at 0x3c073b40 and 0x3c07423c (these two blobs are font descriptors, not text), formats "Version %d.%02d b%d", uses state byte 0x3fccf425 (0 default, 1 or 3 when certain buttons are held, 0xff when done). Strings "Autoplay Mode" and "Init task" belong to it. The "Autoplay Mode" name probably belongs to the startup arpeggio player of round 13 (not confirmed).

Hidden factory test mode (fcn.42026910): menu entries MIDI Test, Pot Test, Encoder Test, Button Test, LED Test, Audio Test, "To start press EDIT", "To quit press ...", "Press all ..." screens (fcn.42026238 to fcn.42026748). Reachable only through the two-flag boot check above.

Buttons: 16 active-low inputs read from a 4-byte shift-register image by fcn.4204f1e0(index): index 0 = byte3 bit0, 1 = byte3 bit1, 2 = byte2 bit0, 3 = byte2 bit1, 4 = byte2 bit2, 5 = byte2 bit3 (by elimination), 6 = byte2 bit4, 7 = byte2 bit5, 8 = byte2 bit6, 9 = byte2 bit7 (by elimination), 10 = byte1 bit2, 11 = byte3 bit4, 12 = byte3 bit7, 13 = byte0 bit7, 14 = byte0 bit2, 15 = byte1 bit7. A bit of 0 means pressed. Whether this index equals the numbering used in BUTTONS.md is not checked.

DIN MIDI: UART1, 31250 baud, 8 data bits, no parity, 1 stop bit, TX = GPIO 17, RX = GPIO 18, 1024-byte RX and TX buffers, no event queue (fcn.42027cec). Output uses uart_write_bytes(1, ...) (fcn.42027fbc for 3-byte messages, fcn.42027fdc for sysex frames).

USB (TinyUSB, fcn.42028050): class-compliant USB MIDI device only. Device descriptor at 0x3c073af4: USB 2.0, class 0, EP0 64 bytes, VID 0x12E6, PID 0x002F, bcdDevice 1.00. Strings: manufacturer "Waldorf Music", product "Protein", configuration "Protein", serial built from the chip MAC (fcn.42032d94). Configuration descriptor at 0x3c073a88: 2 interfaces (audio control, MIDI streaming), bulk OUT 0x01 and bulk IN 0x81, 64 bytes each, one embedded and one external jack per direction, bus powered 500 mA. The PROTEIN mass-storage drive used for updates is NOT part of the app (no mass storage class in the app descriptors).

Firmware update path: the app contains no call to esp_restart, esp_ota_*, esp_partition_write or flash write APIs (searched the decompile). So the app cannot start or perform an update; updating is done entirely by the bootloader (Shift at power-on, PROTEIN drive), as already described in the first section. Consequence for modding: a bad app image does not touch the bootloader; whether Shift at power-on still reaches the bootloader if the app crashes at boot depends on bootloader logic that is not in the files. Unverified, keep as the main hardware risk.

Other system facts: I2S pins MCLK 7, BCLK 9, WS 10, DOUT 8 (round 10); display and LED drivers are on objects 0x3fcd1de4 and 0x3fcd1b78; the display bus setup was not traced (SPI init function has no direct caller found); watchdog and brown-out settings are in the bootloader and sdkconfig, not visible here.

Open: core and priority of each task, GPIO 6 and 21 purpose, which buttons trigger the splash modes and the test mode, display bus pins.

## Friend's QEMU emulator (Claudio_Protein_Emulator.zip, prot-main) - cross-check

What it is: a build and run kit (README, build.sh, run.sh, scripts/, a QEMU patch of 44 KB) that boots PROTEIN.UF2 inside Espressif's QEMU 9.2.2 (ESP32-S3, 8 MB octal PSRAM). It adds I2S audio out, MIDI DIN over a socket, SysEx upload and a small keyboard player (scripts/player.py). Not run here: it needs about 6 GB, a QEMU build and downloads of ESP-IDF v5.5.1 and the toolchain, which the sandbox network list probably blocks. The vendor files (PROTEIN.UF2, Presets_Release_1_0.syx) are not in the zip. Status per its README: sound renders and MIDI/SysEx work; knobs, buttons, encoder and OLED are not modelled; master volume stays 0 until its gain is poked through gdb (set {float}0x3fca4ffc = 1.0 and 0x3fca5000 = 1.0). Needs three small firmware patches for emulation only (scripts/patch.py, which also repairs the image XOR checksum and SHA-256).

Facts it adds that we did not have (its claims, plausible, not re-verified unless marked):
- Partitions: the app looks up two partitions by name: "data" (NVS) and "approm" (type 0x40, subtype 0). Real offsets and sizes are unknown; its emulation table is nvs 0x9000, factory 0x10000 (1 MB, app is about 581 KB), data 0x110000 (1 MB), approm 0x210000 (2 MB) - guesses, not measured.
- Display: SSD1306/SSD1309-class monochrome OLED on GP-SPI2, first command byte 0xAE.
- Pots: 21 pots read with ADC1 continuous (DMA) mode, log text "Pot %d: val %d td: %d". Public teardown reports: 21 knobs, 10 buttons.
- Factory test menu trigger: when all GPIO inputs read 0, the firmware thinks two buttons are held at power-on and enters the test menu (MIDI/Pot/Encoder/Button/LED/Audio). This matches our two-flag boot check in round 16.
- SysEx files: Presets_Release_1_0.syx = 224 sounds x 6 messages of command 0x10; message header after the command: <ref> <count=6> <seq>, sound name starts at byte 9 of message 1. The sysex parser keeps one buffer of at most 1024 bytes, drained once per 64-frame block, so multi-message uploads must be paced.
- Its run reported firmware version "1.03 b3" from the version sysex command while the app descriptor says v1.02; unexplained, not checked.

Corrections to our notes found while cross-checking:
1. Master volume (round 11 and 15 said no reader of engine+0x1688 was found): WRONG. fcn.4200d0d8 (1058 bytes, the float-to-int32 output conversion called from the output loop) loads the value at engine+0x1688 (addmi 0x1600 + offset 136), squares it (mul.s f0,f0,f0) and scales the samples with it before trunc.s. Event type 10 (Master knob, id 13) = master volume, squared gain, ramped through +0x168c. Confirmed in our disassembly at 0x4200d11a to 0x4200d15c. The decompile hid it. The Flavour conclusion (no consumer found) is unchanged.
2. Factory presets (round 14 said the source of the 360 factory presets was unknown): the "approm" partition. AppData_Get fcn.42029f5c does esp_partition_find_first(type 0x40, "approm"), maps it read-only and returns the pointer (cache 0x3fcd45d4, handle 0x3fcd45d0). fcn.42029f08 passes that pointer to fcn.42029e50, which parses a stream of F0..F7 sysex messages, feeds each to the sysex handler and saves it to the next slot. Callers: the "Init Presets ..." controller event 0x1a and first-boot "Init Factory Presets". So factory presets live in approm, outside the UF2. With a blank approm the factory init writes nothing (the unit still plays its init sound). The 12 sysex messages found in DROM (round 14) are something else (init sounds). How approm gets updated is unknown (candidate: a separate UF2 with absolute addresses; unconfirmed).
3. Pin numbers: the README lists I2S and display pins as unrecovered. We have I2S MCLK 7, BCLK 9, WS 10, DOUT 8 and DIN MIDI TX 17 / RX 18 (they confirm 17/18). Display SPI pins still unknown on both sides.
4. Sysex commands 0x11 0x12 0x13 0x14 0x20 0x30 0x40: listed as unreversed by the README; mapped in round 13 (0x14 action still unconfirmed).

Use for the improvement list: the emulator gives a way to test a modified firmware without the unit (boot, notes, MIDI, sound output), which is what the deferred "test firmware" step needed. It cannot show knob, button or display behaviour yet, so improvement 2 (knobs control envelopes) and 4 (time display) still need hardware or more emulator work (ADC and OLED). Its patch.py shows the exact image-fix recipe (XOR checksum slot after the last segment, then SHA-256), which our firmware rebuild tool also needs.

## Effect DSP internals (round 17, pass 1 of the "unmapped" sweep)

All from the decompile, constants read from the binary. No hardware test. Words like "probably" mark things the code suggests but I did not prove.

Shared helpers
- dB to linear `fcn.4200fae8(dB)`: index = (dB + 200) / 300 x 1024 into a 1024-entry table, linear interpolation. The table is built at boot by `fcn.4202b510`: entry i = 10^((i/1024 x 300 - 200) / 20), so -200 dB to +100 dB. Used by drive, EQ and compressor.
- LFO table pointer array at 0x3fcd65d8 (8 bytes per entry: pointer, length). Entry 0 = 1024-entry sine table at 0x3fcd45d8, entry 1 = the dB table at 0x3fcd55d8. Chorus, flanger and phaser read entry 0 when Shape = Sine.
- Shape = Triangle is computed in code: phase p in 0..1, p<0.25: 4p; p<0.75: 1-4(p-0.25); else 4(p-0.75)-1.
- Ramped parameters: each ramp is {current, target, step, count}; the process function advances it once per block.
- Process function = vtable +0x10 of each effect, called per 16-sample block (round 11). Mod inputs (pointers at fixed offsets, default 0.5 or NULL) are added to the knob value before the curve below, so a mod-matrix source can drive these. Which source drives which input: not traced.

Chorus (process `fcn.4200feb4`, 2690 bytes)
- Stages = 2, 4, 6, 8 (enum labels). The process loop runs stage index + 1 times (1 to 4), and each pass handles one left and one right tap, so the label most likely counts left and right taps together (inferred from the loop, not proven).
- Delay buffers: two circular buffers (L, R) of 0x2e87 = 11,911 samples (270 ms) with linear interpolation. Read position is clamped to 11,910.
- LFO rate (Speed) = x^2 x 19.999 + 0.001 Hz (written as divide of phase step by sample rate 44100). Center delay = c^2 x 44100 x 0.2 + 2 samples (up to about 200 ms). Depth: d^3 x 882 samples (20 ms). Voice spread: voice index x Spread x 44100 x 0.02 samples. Shape: 0 Triangle, 1 Sine (table). The L and R LFOs are offset in phase by Spread.
- DryWet uses the curves wet = 2m - m^2 and dry = 2(1-m) - (1-m)^2 (equal-power-like), in the later part of the function.
- Feedback = f x 1.96 - 0.98 (range -0.98 to +0.98). HighCut and LowCut: one-pole filters whose coefficient is sin(x^3 x 0.5 x pi)-style via `fcn.4203d800` (the exact form is from the decompile, details not checked).

Flanger (process `fcn.42012318`, 1631 bytes)
- Speed = x^4 x 20 + 0.001 Hz. Center = c^2 x 1102.5 + 1 samples (25 ms). Depth = d^4 x 1411.2 samples (32 ms). Shape Triangle or Sine (same two LFO forms). PhaseDiff offsets the right-channel LFO. Feedback has a cubic mapping towards 0.45 (constant 0.45 seen).
- Buffer size: 2613 floats per channel (resolved in Round 41, which supersedes the emulator-based flanger findings of Rounds 22e to 22g).

Phaser (process `fcn.42012b18`, constructor `fcn.420132bc`)
- Stages = 2, 4, 8, 12, 16 (enum labels; default index 2 = 8 stages). Model = "Nave" or "PPG" (enum labels; two all-pass coefficient sets). Shape Triangle or Sine.
- Speed = x^2 x 5 + 0.01 Hz. The all-pass tables live at 0x3fca5014..0x3fca541c (RAM). All-pass arrangement per stage not decoded.

Tremolo (process `fcn.42013798`, decompile failed; read from disassembly)
- Shapes: Sine, Triangle, Square. Depth and Smooth: Smooth sets a one-pole smoother of the gain, coefficient 1 / (0.1 + 39.9 x s^2) (constant 39.9 = 0x421f999a, 0.1 = 0x3dcccccd).
- The tremolo has a 32-entry table of doubles at 0x3c071960: 1024, 768, 512, 384, 256, 192, 128, 96, 64, 48, 32, 24, 16, 12, 8, 6, 4, 3, 2, 1.5, 1, 0.75, 0.6667, 0.5, 0.375, 0.3333, 0.25, 0.1875, 0.1667, 0.125, 0.0938, 0.0833. Looks like note-length divisions (tempo sync rates). Only this function reads it. Whether the Speed knob uses it always or only with a sync mode is not confirmed.

Drive (process `fcn.42011924`)
- Types: PNP, Tube, PickUp, Diode, Crunch (enum labels; the five waveshapers are applied by `fcn.42011768`/`fcn.42011584`, not decoded further; round 8 describes the same shapers for the voice).
- Gain = x x 40 - 20 dB (via the dB table), applied before the shaper to both channels. Amount (squared into the shaper parameter). DryWet crossfade 2m - m^2 on the shaped signal. After the shaper a hard clip stage `fcn.4204dd30` runs.

EQ (coefficients `fcn.42011d80`, process `fcn.42012058`)
- Four bands: low shelf (`fcn.4202a308`), two peak bands (`fcn.4202a6bc`), high shelf (`fcn.4202a4b4`). Sample rate 44100.
- Frequency = 2^e Hz with e linear in the knob: low shelf e 4.907..8.814 (30 Hz to 450 Hz); mid 1 and mid 2 e 7.644..12.773 (200 Hz to 6.9 kHz); high shelf e 10.551..14.288 (1.5 kHz to 20 kHz). Peak Q = 2^(q x 7.322 - 3.322) (about 0.1 to 10). Gains are +-20 dB, scaled by the EQ "Gain" knob (EQ Gain scales all band gains: (g - 0.5) x 40 dB x DryWet). Band gains: (value - 0.5) x scale x 40 dB.
- A mod offset (default 0.5, subtract 0.5) shifts all four frequencies in the exponent domain (read at obj+0xf4 and +0x144). Coefficients are only recomputed when the flag at obj+0x13c is set.

Compressor (control `fcn.4200fb78`, kernel `fcn.4202a008`)
- InGain and OutGain = x x 60 - 30 dB. Threshold = (1 - t) x -50 dB (0 to -50 dB). Ratio = r^3 x 99 + 1 (1:1 to 100:1). AutoGain: when on, extra make-up gain computed from threshold and ratio (formula in the kernel, not worked through).
- Kernel: a peak detector over 4-sample groups (max abs over L and R), one-pole attack/release on the detector (coefficients obj+4 / +8), subtraction of the threshold, a second smoothing of the over-threshold amount (obj+0x10 / +0x14), then gain = ((thr + over) / thr)^(1 / ratio - 1). With LookAhead the audio is read from a delay line (obj+0x32c above 3 samples), otherwise the gain is applied directly.

Delay (process `fcn.42010ba0`, 2012 bytes)
- Two circular buffers of 0x6baa8 = 441,000 samples each (10.0 s at 44.1 kHz), linear interpolation. Delay time is clamped to 1..88,198 samples (2.0 s).
- Free time: t = x^2 x 88,198 + 1 samples. Synced: samples = ratio x beat length, ratio from the table at 0x3c06ffd0 = 0.125, 1/6, 0.1875, 0.25, 1/3, 0.375, 0.5, 2/3, 0.75, 1, 1.25, 4/3, 1.5, 5/3, 1.75, 2 (labels in the UI: 1/32, 1/16 tri, 1/32 dot, 1/16, 1/8 tri, 1/16 dot, 1/8, 1/4 tri, 1/8 dot, 1/4, 1/4+1/16, 1/2 tri, 1/4 dot, 1/4+1/4 tri, 1/4 dbldot, 1/2). Beat length comes from a double in the transport record (tempo; see round 13).
- Time changes are slewed: the read pointer moves by (target - current) x 0.000125 per sample (about 8000 samples to arrive), so a time change pitch-bends the echoes (tape-like).
- Link on: right time follows the left. Feedback path has HighCut and LowCut one-pole filters. DryWet uses 2m - m^2 curves.

Reverb (control `fcn.42013420`, core class from `fcn.4202ad50`)
- One shared circular buffer of 65,536 samples (index masked with 0xffff). Pre-delay = x x 16,370 + x^2 x 16,370 samples, clamped to 0x7fe0 = 32,736 and rounded down to a multiple of 16 (so up to 0.74 s).
- Delay lengths set by the constructor (samples): input diffusion allpasses 206 (0xce), 365 (0x16d), 915 (0x393), 1333 (0x535), 2307 (0x903); tank delays 8764 (0x223c), 11374 (0x2c6e), 16768 (0x4180), 24200 (0x5e88), 28051 (0x6d93) and others (0x46bd, 0x7ff2 ...); output tap delays listed from obj+0x280 for left and right (7 taps each: 0x482a, 0x577d, 0x695e, 0x78e1, 0x1449, 0x234b, 0x3278 ... for one side and 0xb02, 0x1d8e, 0x2931, 0x3b92, 0x529a, 0x606f, 0x6e42 for the other). That is the usual shape of a plate or "Dattorro"-style tank, but I did not check the topology against a reference, so treat that label as a guess.
- ModRate = x^3 x 20 Hz (the constant 20.0 with x^3); ModDepth scaled by 5. Color: coefficient pair derived from (value x 2 - 1) with limits 0.99, 0.995, -0.99 and slopes 1.69/1.89 (a tilt filter; details not decoded). Time scales the feedback gains of the tank.

Output stage and master (fcn.4200e700 tail, FX base constructor fcn.4200dd84)
- FX base constructor values: limiter gain state +4 = 1.0, attack coefficient +8 = 0.2, release coefficient +0xc = 0.002, threshold +0x10 = 0.85, gain smoothing +0x14 = 0.01. Both FX slots start pointing at the Bypass object (+0x94).
- Limiter, per group of 4 samples: peak = max |x| over both channels; env += (peak - env) x (0.2 if rising else 0.002); target gain = 1.0 if env <= 0.85, else 0.85 / env; gain += 0.01 x (target - gain); the gain multiplies both channels. So linear threshold 0.85 (about -1.4 dBFS).
- The "master filter" biquad at fx+0x60..0x80: constructor sets b0 (+0x68) = 1.0 and every other coefficient 0, so as initialised it is a pass-through. I found no other writer of these coefficients (unconfirmed, a computed-pointer write could exist).
- Final safety stage `fcn.4200d938` (16 samples): +-infinity becomes 0; otherwise clamp to [-1, +1]. NaN falls through the comparisons and ends up -1.0 (from the decompile, not tested).
- Conversion to 32-bit integers is in `fcn.4200d09c` (round 11).

DirtType and noise generators (found: they are inside the voice render fcn.4037bfb0, not in a separate function)
- DirtType value d = (int)(value + 0.05) (enum: 0 Static, 1 Crackle, 2 Geiger, 3 Click, 4 Burst 1, 5 Burst 2, 6 Burst 3, 7 Ring Mod). Amount = NoiseVolume^2 (ramped, round 8) x 2^-34 x 2^32-range random, so about +-0.125 x amount around zero, added to the 16-sample block after the oscillator mix.
- Random source: 32-bit LCG state at voice+0x160, x = x * 0x41c64ed1 + 0x329d (one step per use).
- 0 Static: a fresh random number every 4 samples (4 distinct values per block, each held for 4 samples).
- 1 Crackle: per 4-sample group, an impulse when the top byte of the random number equals 0x80 (about 1 in 256).
- 2 Geiger: same, but the test is top 12 bits equal 0x800 (about 1 in 4096), so far rarer.
- 3 Click: one impulse (value from a stack variable, amplitude = amount) on the last sample of the block when the counter at voice+0x5c4 is 0, i.e. at note start.
- 4/5/6 Burst 1/2/3: while the counter voice+0x5c4 is below 350 / 1000 / 10000 respectively, per 4-sample group an impulse with probability 1/16 (top nibble 0xe). The unit of the counter (samples or blocks) is not confirmed.
- 7 Ring Mod: out = vol2 x s2 + vol1 x s1 + ring x s1 x s2 x 1.953e-5 (round 8).

## Screens and drawing (round 17)

Display
- 128 x 64 monochrome, 1 bit per pixel, framebuffer 1024 bytes. Text/glyph blit `fcn.42028e0c` writes into a buffer with column-major layout: byte index = x x 8 + (y >> 3), bit = y & 7, x clamped to 127 and y to 63. Fonts are objects with a vtable: +8 height, +0xc glyph width for a char, +0x10 pixel column bits for (char, column), +0x14 spacing. `fcn.42028f30(x, y, w, h, text, inverse)` draws a label box (button style, e.g. " Actions ", "Edit" at 0x7f x 0x20); `fcn.42028884` transposes 8x8 bit blocks (page layout conversion) just before the SPI transfer.
- Transfer `fcn.420273f0`: two 3-byte command writes (words at 0x3fca0544, 0x3fca0548), then the 1024-byte frame copied to 0x3fcd2e60 and sent as 0x2000 bits over SPI (`spi_master_7`). Brightness `fcn.42027360`: sends command 0xd9 (precharge) with a value from a 20-entry table at 0x3c067600 and command 0x81 (contrast) with value index x 16 + 1 (set to 0xff when the setting is 15 or more). These command bytes match common 128 x 64 OLED controller commands (SSD1306 family), but the exact chip is not confirmed.
- Screen dump over sysex (command 0x30, round 13) sends this framebuffer.

UI structure
- `Event__UI_INFO_MSG` 0x4201d680 (misnamed by the log string) is the central UI event dispatcher. Event types in: 0 button, 2 pot, 3 encoder, 5 info message (text copied to ui+0x16bc, 0x100 bytes; shown until dismissed or 10 minutes, deadline = now_ms + 600,000), 7 parameter update from the engine (value x 1e-6 into the UI mirror at ui+0x9a4), 12 and 13 set a flag at ui+0x28. Each event goes to the page objects in this order: popup/message layer at ui+0x2c, ui+0xb8, the front panel (`fcn.4201e640`, ui+0x198), ui+0x534 and 0x548 and 0x584, the sound list at ui+0x5ac, 0x75c, 0x780, 0x790, 0x79c, 0x7ac. Which object is which page is only partly resolved (below). Page vtables: 0x3c0736ac, 0x3c073634, 0x3c073788, 0x3c073854, 0x3c0737ac, 0x3c0738c8, 0x3c073688.
- Popups: `fcn.4201d2dc(ui, text, ms)` shows a timed message (e.g. "Saved", "Copied!", "Pasted!", "Initialized!", "Empty", each 2000 ms).
- Pages and the functions that draw or handle them (string anchored, descriptions from strings and calls; items marked ? are guesses):
| function | page text |
|---|---|
| 0x4201dad0 | "Waiting for CC" with " Cancel " (MIDI learn prompt for Map CC) |
| 0x4201dbf0 | "CC Mappings" list ("%d", "%c: %s", " Delete ", "Not Mapped", "Nothting Mapped" - the typo is in the firmware) |
| 0x4201dd54 | "For CC %d" with Delete / Cancel, title "Delete Mapping" |
| 0x4201dee4 | the popup "%d -> %s" shown after a CC was learned (3 s, round 13) |
| 0x4201df88, 0x4201e108 | front-panel section constructors (Wave/Filter/Amp/Mod/Pan/Master, round 5 notes) |
| 0x4201f204 | envelope view with "Att/Dec/Sus/Rel" |
| 0x4201f95c | "Osc 1 - Details" / "Osc 2 - Details" with "Link", "Close" |
| 0x4201ff20 | "Dirt" and "Amount" display |
| 0x420200cc | "Vel Amnt" |
| 0x4202037c | "Chord" |
| 0x420206f0 | "Pitch - Details" (Osc 1, Osc 2, Pitch Bend, Mode) |
| 0x42020c30 | "ModMatrix" with "LFO 1 - Details", "LFO 2 - Details" (Shape ...) |
| 0x4202121c | "[=]" marker row |
| 0x420213e0 | "FX" |
| 0x420214c8 | Master page: "Flavour" and "Volume" with "%.02f" |
| 0x420216d0, 0x420217b0 | sound list: "%03d %s", "%03d [empty]", " Load ", " Actions " |
| 0x420218ac | handler of the Actions menu choices (below) |
| 0x42021b0c | confirm dialog "Please Confirm!" "Init Sound" (Ok / Cancel) |
| 0x42021d00 | arp/play page: "Play", "Note", "Octs" |
| 0x4202223c, 0x420226c8, 0x42022834, 0x420228c0, 0x42022908 | sequencer page: "Sw %d" (switch/step), "Gt %d" (gate), "Record", "Edit", "Recording...", "%d steps", " Stop " (log lines "Start Record", "Stop Record") |
| 0x42022d98 | value readout "[%d]", "%.0f%%" |
| 0x42023088, 0x420231cc, 0x4202330c | save page: "[Name]", " Save ", "Saved", "Edit Name" (name editor `fcn.42024708`/`fcn.42024828`, character set "ABC...xyz0123456789-_!()[]=.#+*?% " and a back marker "<") |
| 0x420233a4 | "[empty]" slot entry |
| 0x420234ac, 0x420236ac | layer pages: "Layer %c", "(Layer %c)", "Vol", "Active", "Actions", " Select ", "Layer", " %c ", " Load " |
| 0x42023ee8, 0x42023f30, 0x42023f90 | "Copied!", "Pasted!", "Initialized!" |
| 0x42024090 | page header line: "Multi Mode" when LayerMode = 4 (also shows the LayerAlloc value), else "Single Mode" |
| 0x420244c8 | two debug-looking lines "0: %.1f < %.1f" and "1: %.1f < %.1f" read from the engine; purpose unconfirmed |
| 0x42024b88 | "->" row (mapping view?) |
| 0x42025074 | a status/diagnostic task (big function): logs "Core 0: avg ... us max ... us | Core 1: ..." and "Sync %s bpm %.1f"; see the "running" / "idle" strings |
| 0x42026198 | boot/version screen "Protein", "Version %d.%02d b%d" |
| 0x420262b0 ... 0x42026910 | factory test screens (round 16): LED test ("Check all LEDs"), "To quit press EDIT", button/pot/encoder tests, "Connect IN->OUT" MIDI test, menu "MIDI Test / Pot Test / Encoder Test / Button Test / LED Test / Audio Test" |
- Menu label arrays in RAM (0x3fca04d4..0x3fca0530):
  - Global menu: Info, Map CC, Mappings, Close.
  - Sound list " Actions " menu, 12 labels: Filter (Category Filter), Delete, Delete Range, Send, Send Range, Start Receive, End Receive, Copy, Paste, Close, Load Factory. Handler `fcn.420218ac` by item index: 0 category filter dialog; 1 Delete Preset confirm (slot passed through a string helper); 2 Delete Range dialog; 3 sysex "dump sound" (controller event 0x13); 4 Send Range dialog; 5 Start Receive dialog ("Receive Presets", "Store at:"); 6 controller event 0x18 (End Receive, so "end of batch"); 10 controller event 0x1a = "Init Presets" = Load Factory. The index to label match is inferred from order and from the called events, not from a table that pairs them. Indices 7 to 9 (Copy, Paste, Close) return from this function; Copy and Paste are handled elsewhere (the "Copied!" "Pasted!" popups).
  - Layer menu: Init Layer, Load Layer, Copy Layer, Paste Layer, Close (two shorter variants exist).
- Dialog strings found: "Category Filter", "Set", "Delete Preset", "Delete Presets", "Delete Range", "Send Presets", "Send", "Send Range", "Store at:", "Receive Presets", "Start Receive", "End Receive", "Load Factory", "Init Sound", "Empty".
- Info popup text itself ("Info" in the global menu) is not a page of its own that I found: Info shows through the event type 5 message path above (text up to 255 characters). What fills it is unconfirmed.

## Preset matching, sysex 0x14 and other small gaps (round 17)

Preset load (layer copy) `fcn.420298f4(sound, slot, srcLayer, dstLayer)`:
- Reads the NVS blob of the slot (`fcn.42027a7c`, key "sound-%03i"), checks that the first 4 header bytes are not 0xFF (empty), then walks the 8-byte records after the 32-byte header: `fcn.42018bc4(tag)` finds the parameter index whose tag equals the stored 4-byte tag (linear scan over the 0x231 parameters; for the 0x150 per-layer parameters the tag is 3 name characters plus the layer digit '1'..'4', so layer digit is part of the key). Only records belonging to srcLayer are taken, and the index is moved to dstLayer (`index - src x 0x54 + dst x 0x54`). The value goes through `fcn.420294f0(sound, index, value)` (a version/sanity conversion, below) and is stored as a float in the sound's parameter array; the name (24 bytes) is copied to sound+0x8c4 and the used flag/category byte to +0x8dd = 8.
- Missing blob: the sound is filled with defaults (`fcn.42018d68` per parameter), category/flags set to 0x800 and the name "Init".
- `fcn.420294f0` is a value migration: it adjusts stored values for some per-layer parameter slots (index mod 0x54): slot 4 (Wtb1Table) subtracts 24 when the value is above 0x37; slot 0x2b and 0x30 pass through `fcn.4203d928`; slots 0x3a..0x4f in groups of 3 (mod-matrix style entries) add 1 or 2 when the value passes thresholds 5, 9/10, 11/14 (inserting enum entries that older presets did not have); slots 0xb..0x28 map a 0..1 value through a 128-entry table at 0x3c075f30 in some cases. It has a mode argument (the old format version, header byte 0x19) that the decompiler dropped; the per-version table is in Round 36. In short: presets from older firmware are converted when loaded.
- Save direction `fcn.42029c10`: copies the 32-byte header and writes {tag, float} pairs for all 0x231 parameters into the snapshot buffer (round 14). The sysex sound dump `fcn.4201af24` sends the same pairs: 6 messages, up to 98 pairs per message, each tag and each value as five 7-bit bytes (a 32-bit word split 4 + 7 + 7 + 7 + 7); first message carries name and flags; the second header byte of each message is the total (6), the third is the 1-based index. This explains the embedded factory messages starting F0 3E 2F 00 10 xx 06 xx.

Sysex command 0x14 (`MidiSysexController__processMessage`): not a select command. It reads a 14-bit slot number (bytes 10 and 11, high 7 bits first). If the slot is above 359 it answers status 2 (bad index); if the slot is empty it answers status 1; if the slot is used it answers status 0. No state change. So 0x14 = "query whether preset N is used" (confirmed from code).
Sysex 0x20 (preset names): the start index and count are 14-bit values; count 0 or above 360 answers status 6.

LayerAlloc labels (resolved): 0 Layered, 1 Round Robin, 2 Random Robin, 3 Midi Split (enum table at 0x3c072cd8..). LayerMode labels continue as Solo A, Solo B, Solo C, Solo D, then Multi. This matches the note routing rules in round 5.

Other labels found: Chorus/Flanger/Phaser Shape = Triangle, Sine; Chorus Stages 2, 4, 6, 8; Phaser Stages 2, 4, 8, 12, 16; Phaser Model = Nave, PPG; Tremolo Shape = Sine, Triangle, Square; Drive Type = PNP, Tube, PickUp, Diode, Crunch.

Still not found: what sets the clock-enable byte at +0x220 that the clock handler `fcn.4201beac` tests (the only writes by literal offset belong to the Globals object, which is a different object with the same offset). With Sync Source = Extern (round 12) this is the likely switch, but the link is unconfirmed.

## Second pass: checks against the emulator and corrections (round 18)

Method: the friend's QEMU emulator (see the cross-check section) was driven from scripts in `scripts/emu_tests/` (they need the Windows kit's Linux side or any Ubuntu 24.04 install built from it). They load an edited sound over MIDI sysex, play a note, record the audio to a WAV file, and measure it. Results below are emulator results: the firmware logic is sample-count driven, so timing in samples is exact for this firmware; only the assumed sample rate (44,100 Hz, see below) turns it into milliseconds.

Sysex sound dump format (confirmed with the real receiver `MidiSysexController__onSoundDump`, and by loading sounds in the emulator):
- Message: F0 3E 2F devID 10, then m[5] = transfer id, m[6] = total number of messages, m[7] = message number (1-based), m[8] = (first message: a flag byte; later messages: a byte whose meaning I did not work out), then data, then a checksum byte, then F7.
- Checksum = (sum of bytes m[1] .. m[len-3]) & 0x7f, stored in m[len-2]. A wrong checksum is answered with status code 3 and the message is ignored. Verified on all 12 embedded messages.
- The transfer id (m[5]) of messages 2 to 6 must equal the id of message 1; message 1 sets it. The message number must count up by one, otherwise status 4 or 5.
- First message: m[9] and the name start at m[10] (name field runs to m[33]), pairs start at m[34]. Later messages: pairs start at m[8]. Each pair = 5 bytes tag + 5 bytes value; a 32-bit word is sent as 5 bytes [bits 31-28, 27-21, 20-14, 13-7, 6-0] (first byte has only 4 bits). The tag is four ASCII characters (for example "o1s1" = Osc 1 semitone, layer 1; "fxt1" = FX 1 type; "dldl" = Delay Left). Values are IEEE float32 words. Up to 98 pairs per message; the dump has 6 messages.
- Values are the stored raw values: discrete parameters are the step number as a float (fxt1 = 8.0 is Delay), continuous ones are 0..1.
- The two embedded sounds in the firmware image (0x3c076130 area) are old-format dumps with 536 pairs, not 561, and have transfer id 1. A receiver must be sent id 0 or any equal id; the embedded copies work after fixing the checksum.
- The embedded default sound has FX1 = Delay (synced, 1/8, feedback 0.75, DryWet 0.225) and FX2 = Reverb. LayerAlloc etc. not part of it.
- Tags for the global parameters 0x150..0x19f: fxt1 fxt2 bydm; chorus chdw chdp chsp chfe chce chsd chsh chst chhi chlo; flanger fldw fldp flsp flfe flce flpd flsh; phaser phdw phdp phsp phfe phce phsd phpd phsh phst phmo; tremolo trdw trsp trpd trsh trsm; drive drdw dram drga drty; compressor cpra cpth cpig cpog cpag cpat cprl cpla; EQ eqdw eqga eqlf eqlg eqmf eqmg eqmq eqnf eqng eqnq eqhf eqhg; delay dldw dldl dldr dllk dlsy dlrl dlrr dlfb dlhi dllo; reverb rvdw rvti rvco rvmd rvmr rvpd; then asmd (arp/seq mode) aral aroc arre rhpt ... Per-layer tags (index 0..83, the last character is the layer digit): o1s o1d o1k o1v w1t w1p w1l o2s ... dty nsv gli glm pbr, wea wed wes wer (wave env A D S R) w1e w1v w2e w2v, fcu fre fty fkt drt (filter and drive), aea aed aes aer (amp env), ave, fea fed fes fer (filter env), fce fve, l1p..l2y (LFOs), lap lrp lav lds (layer pan, ..., volume, destination), plm, ms1/md1/ma1 .. ms8/md8/ma8 (mod matrix source, destination, amount), mnp, act (Active in Multi).

Envelope update interval (the open question from round 10 and round 6): measured.
- Method: part A amp envelope with attack 0, decay 0, sustain 1, release R = (idx + 0.5) / 127 (so the engine index int(R x 127) = idx), no effects, note held 0.6 s then released, WAV recorded, 5 ms RMS envelope, a straight line fitted to the decay between -10 dB and -35 dB.
- The amp stage is squared in the output: the final gain is 1.2 x (level x velocity term)^2 x ... (voice render, last section), so audio amplitude falls with level^2 (twice as many dB per update as the level itself). With coefficient c[idx] from the ROM table (round 6), dB per update of the level = -20 log10(1 - c).
- Measured slope and the resulting interval per envelope update if the amplitude were linear in the level (k = 1), then corrected for the square (T = 2 x that):
  | idx | c | measured slope | k=1 interval | corrected (k=2) |
  |---|---|---|---|---|
  | 30 | 0.00839 | -112.4 dB/s | 0.651 ms | 1.30 ms |
  | 40 | 0.00420 | -58.1 dB/s | 0.628 ms | 1.26 ms |
  | 50 | 0.00208 | -26.4 dB/s | 0.684 ms | 1.37 ms |
  | 60 | 0.00105 | -17.7 dB/s | 0.514 ms | 1.03 ms |
  | 70 | 0.00052 | -9.6 dB/s | 0.472 ms | 0.94 ms |
- Mean about 1.18 ms, scatter about +-20 percent. This fits the 49-sample control tick (1.111 ms) and rules out the other two candidates from round 10: every 16-sample block (0.363 ms) would give 0.73 ms corrected and every 8th tick (8.89 ms) would give about 17 ms corrected. Conclusion: envelope stages update once per control tick, 49 samples = about 1.11 ms, not once per 8 ticks as I read round 10. The `(tick count & 7)` gate I found there belongs to something else (probably the LFO products); the stage-machine gate "once per 8 render blocks" in round 6 does not fit the measurement either (that would be 2.9 ms). The measured value is also a little slower than 1.111 ms in most rows; the cause is not known.
- Consequence for improvement 4 (ms display): time constant of one stage tau = 1.111 ms / c[idx]. Time for the LEVEL to travel 99 percent of the way = 4.605 x tau; for the audio (level squared) to fall 40 dB in release = 2.303 x tau. Examples with 1.111 ms: idx 20 (c 0.0168) tau 66 ms; idx 40 tau 265 ms; idx 60 tau 1.06 s; idx 70 tau 2.1 s; idx 100 (c 6.5e-5) tau 17 s. Measured -40 dB release at idx 40 was 0.76 s against 0.61 s predicted. This is no longer blocked; see the refinement below. Treat the numbers as plus or minus 15 percent until measured on the unit.

Envelope timing, refinement (all three stages, emulator, neutral test sound):
- Setup fix: the embedded test sound has the arpeggiator on (asmd = 2), which retriggered the note about every 128 ms and made the first attack and decay runs look flat and erratic. With asmd = 0 (off) the curves are clean. Other settings: effects off, filter open, filter and wave envelopes neutral, one note on the amp envelope, runs done one emulator at a time.
- Release (5 runs, fit of ln amplitude against time, corrected for level squared): T = 1.18, 1.08, 1.10, 1.18, 1.20 ms for idx 30, 40, 50, 60, 70; mean 1.15 ms.
- Decay (sustain 0.3, idx 30, 40, 50, 60): T = 1.1 ms in all four, residual 0.03 to 0.07, the cleanest result. The decay is a one-pole fall towards the sustain level.
- Attack (idx 20 to 60): the curve is a one-pole rise towards 1.0 (fitted target 1.00). Fitted time constants 57, 113, 246, 450, 938 ms against 66, 132, 265, 534, 1059 ms predicted from 1.111 ms / c, so about 0.84 to 0.93 of the prediction (mean 0.87, an apparent step of about 0.97 ms). The slower attacks have a noisy plateau reference, so the fit error is larger there; cause of the 13 percent difference not known (could be the way I set the start of the curve).
- Result: release, decay and attack all agree with one update per 49-sample control tick (1.111 ms) within about 15 percent. The firmware's own sample-count logic makes the true value exactly 49 samples at 44,100 Hz if the gate reading is right; the measured scatter does not distinguish between that and something close to it.
- Still unconfirmed: the delay stage (stage 1) was not measured; behaviour with the +-32 jitter on the tick threshold (part setting above 0.25) was not tested; nothing is measured on real hardware.

I2S sample rate: the emulator derives the rate from the clock divider registers the firmware programs (source / divider / bit clock divider / slot size) and only falls back to 48 kHz if the result is out of range. The recording came out at 44,100 Hz, so the real I2S configuration yields 44,100 Hz (closes the open point in round 10).

Delay time check: Left = Right = 0.5, free time, no feedback, dry/wet 0.5: the second note edge arrived 515 ms after the first (formula x^2 x 88,198 + 1 samples = 22,050 samples = 500 ms). So the formula is right to within 3 percent; the 15 ms extra is not explained (could be the 16-sample block pipeline plus the edge threshold of my measurement).

Corrections to earlier rounds
- Round 10 "Open points": envelope update interval is about 1.11 ms (see above); I2S rate is 44,100 Hz.
- Round 11: the master biquad is initialised as a pass-through (b0 = 1, others 0); limiter values and the final safety clip are now known (round 17).
- Round 13: sysex 0x14 is "is preset N used" (status 0 used, 1 empty, 2 bad index), not a select; the sound dump format is now documented above.
- Round 14: "the tag/value pairs are matched back to parameters (matching code not traced)": traced, see round 17. The load path also converts old-format values (`fcn.420294f0`).
- Round 8/9: the DirtType generators are in the voice render, see round 17.
- LayerAlloc display names: Layered, Round Robin, Random Robin, Midi Split.
- The 'unconfirmed' envelope timing table in round 10 is superseded by the measurement above (the 1.11 ms column).

Still open after both passes
- Phaser all-pass layout and the exact Model difference; flanger and chorus feedback and filter details; reverb topology versus a known design; tremolo speed mapping to the tempo table; the compressor AutoGain formula.
- Which UI object is which page for ui+0x534/0x548/0x584/0x75c/0x780/0x790/0x79c/0x7ac; what the Info menu entry shows; what the debug lines in `fcn.420244c8` show. (resolved: Rounds 25b, 25c and 39)
- `fcn.420294f0` migration modes; what sets the MIDI clock-enable byte; the unexplained 15 ms in the delay time and the 10 to 20 percent slow bias in the envelope measurement.
- Attack and decay time not measured separately; no hardware measurement of anything.

Addendum to the display notes: the 20-entry table at 0x3c067600 is 31, 47, 63, 79, 95, 111, 127, 143, 159, 175, 191, 207, 223, 239, 255, 251, 247, 245, 243, 241 (indexed by Display Brightness 0..19, sent with command 0xd9).


## Round 19: emulator front panel (sliders, buttons, encoders, screen)
All of this was tested in the emulator only; nothing is checked on hardware.

Photo match (panel photo from the user). 15 black/white/blue pots, 10 buttons and 6 encoders, matching the firmware counts (15 pots, 16 button bits = 10 buttons + 6 encoder presses, 6 encoders).
- Pots (id: label): 0 Detune/Chord, 1 Glide, 2 Wave/Level, 3 Cutoff/Track, 4 Resonance/Dirt, 5 Envelopes Wave, 6 Envelopes Filter/Pan, 7 Wavetable Env Amnt/Vel, 8 Filter Env Amnt/Vel, 9 Envelopes Amp/Volume, 10 Speed/Ratio, 11 Mod Amount, 12 FX Amount/Value, 13 Volume, 14 Flavour. (Volume and Flavour are pots, not encoders.)
- Encoders: 0 Select, 1 Semitone/Scale, 2 Table, 3 Target (mod destination), 4 Source/Shape (mod source; with shift the LFO shape, which matches the firmware), 5 Type/Param. Presses are button ids 10 to 15 (10 + encoder); ids 13 to 15 have no handler.
- Buttons: 0 to 3 Layer A to D, 4 Multi, 5 Shift, 6 Settings, 7 Edit/Init, 8 Play/Arp-Seq, 9 Presets/Save. Ids 6 to 9 are matched to labels by function (unconfirmed).

Input hardware as the firmware sees it:
- Buttons and encoders come from a 4-byte input shift register read by a bit-bang loop on GPIO 11 to 14 (`fcn.42028884`, object pointer at Board+0x12a8; bytes 0 to 3 = inputs, bytes 4 to 7 = LED outputs). Buttons are active low. Bit map (byte, bit): ids 0 (3,0), 1 (3,1), 2 (2,0), 3 (2,1), 4 (2,2), 5 (2,3), 6 (2,4), 7 (2,5), 8 (2,6), 9 (2,7), 10 (1,2), 11 (3,4), 12 (3,7), 13 (0,7), 14 (0,2), 15 (1,7). Encoder 2-bit states (low bit, high bit): enc0 (1,0)(1,1), enc1 (3,2)(3,3), enc2 (3,5)(3,6), enc3 (0,5)(0,6), enc4 (0,0)(0,1), enc5 (1,5)(1,6).
- Encoder decoding table at 0x3c0739d0 (16 signed bytes, index previous x 4 + new). Clockwise from the idle state 3 is 3, 1, 0, 2, 3 (four steps = one click, +4); counter-clockwise is 3, 2, 0, 1, 3.
- LED bytes (obj+4..7), bit per LED id from `fcn.4204f3ac`: ids 0 to 8 = byte 6 masks 08 04 02 01 20 10 80 40 and byte 7 mask 80; ids 9 to 15 = byte 7 masks 01 02 04 08 10 20 40. Which LED is which button is not confirmed.
- Pots: ADC continuous mode (not modelled by the emulator). `fcn.420283d4(board, pot)` returns millivolts; `fcn.42027278` converts to 0-511 (below 1680 mV: (mV-10)x255/1670; above: (mV-1680)x256/1370+255) with a hysteresis of 13 mV.
- Display frame buffer: 1024 bytes at 0x3fcd2e60, byte = x x 8 + (y >> 3), bit = y & 7 (readable through the gdbstub; confirmed by rendering it).

Pot writer touch logic (`fcn.4201ea04`, confirmed by tracing): the first reading after 0.5 s without a call only sets a baseline (stored with the parameter index); later readings are acted on only if they differ from the previous by more than 30 counts, or come within 99 ms of the last sent event. A single jump of a knob after a pause is therefore ignored. Settings word 8 (address 0x3fccfd34) = Pot mode: 1 Relative (default), 0 Absolute. The emulator panel pokes 0.

Emulator mechanism (patch.py, "PANEL" entries): `fcn.420283d4` is replaced by a 20-byte stub returning the u16 at 0x50000000 + 2 x pot; the first part of `fcn.42028884` after its entry instruction is replaced by a 47-byte stub copying the 4 mailbox bytes at 0x50000040, inverted, into the input bytes (so an empty mailbox means nothing pressed). 0x50000000 is RTC slow RAM, zero at boot and unused by the firmware. LED outputs are no longer shifted out (they stay in the object). The stubs were assembled with xtensa-esp32s3-elf-as.

gdbstub notes: connecting to the QEMU gdbstub pauses the guest. Detach with a multiprocess-style handshake and `D;1` (as the real gdb does); a bare `D` leaves the guest paused or the audio silent. panelio.py does one short session per operation.

Sysex 0x11 (request current sound) works as F0 3E 2F 7F 11 00 F7: the synth answers with the 6-message sound dump (561 pairs). Used by scripts/emu_tests/readback_sysex.py to read parameters back. Results: slider on Cutoff set fcu1 to 0.196, 0.587, 0.978 for slider values 100, 300, 500; Wave slider set w1p1 to 50 (raw step number); Table encoder +3 clicks gave w1t1 = 2, then -1 click gave 1; Semitone encoder +2 clicks gave o1s1 24 to 25 (the first click of an encoder seems to be used for opening its page; unconfirmed).

Files: scripts/panelio.py (gdbstub client, control model), scripts/panel_ui.py (Tk panel), scripts/player.py (now embeds the panel; the old gdb gain poke is gone, Volume is pot 13), scripts/patch.py (PANEL patches).

Slider sync (round 20): the panel keeps the sliders in step with the sound. player.Emulator keeps what the synth sends on MIDI out; request_sound() sends the 0x11 request and parses the six messages into {tag: value} (panelio.parse_sound_dump). panel_ui.slider_positions maps pot to tag with the layer digit (part + 1): 0 o1d, 1 gli, 2 w1p (step number / 64), 3 fcu, 4 fre, 5 wea, 6 fea, 7 w1e, 8 fce, 9 aea, 10 l1p, 11 ma1, 14 flav; continuous values x 511. Checked by moving each pot and diffing two dumps: every one of those tags changed, and after Layer B the same pots changed the "2" tags. The part byte is read at *(0x3fcd0104+0xd4)+0x9a0 (confirmed: 0 then 1 after Layer B). Selector bytes for envelope slot (P+0xa0/0xf8/0x170 +0x2c), LFO (P+0x1a0+0x34), mod slot (P+0x1a0+0x30) are read but only value 0 was observed, so their meaning is unconfirmed; the oscillator selector is fixed at osc 1. Pot 12 (FX) is not synced. Sliders are set without sending pot events.

LED map (round 20, from emulator tests: clean boot, one control at a time, LED bytes sampled 40 times). LED object entries are 16 bytes at Board+0x16c (0x3fcd1ce4 + 16 x id): +0 mode (0 off, 1 on, 2 blink), +4 id, +6 blink period ms (fcn.4202722c). Results (id: meaning): 1, 3, 5, 7 = selected layer A, B, C, D (exclusive); 0, 2, 4, 6 = layer A, B, C, D active in Multi (lit by act1..4 = 1 in a loaded sound; only A is on by default); 9 = Multi; 10 = Shift (latches); 11 = Settings; 12 = Edit; 13 = Arp/Seq page (Shift+Play); 14 = Presets (on by default, blinks in the Save prompt, Shift+Presets); 15 = LFO (written from the LFO output at 0x3fcd1dd4 = entry 15, seen blinking in every test). Plain Play changed no LED. LED 8 never lit: not Play, not arp modes 1 to 3, not a held note with arp on. Unknown; may be unused. The label match for 11 to 13 depends on buttons 6 to 8 being Settings, Edit, Play (by function, unconfirmed). panel_ui.py now labels the LED row with these names.

LED 13 correction (round 20): sampling the LED state every few ms around a Play press (button 8) shows LED 13 on and LED 14 off for as long as Play is held, back to 14 on release; same with the arpeggiator off, mode 2 with a held note, and mode 3. Shift+Play leaves 13 on (latched) and 14 off. So 13 is the Play LED, not only an Arp/Seq page indicator. Colour cannot be seen from the firmware (each LED is one on/off bit), so whether it is blue on the unit is unconfirmed. LED 8 still never lit.

## Round 21: sound engine pass (names, filter, mod matrix, layer allocation)

Step names are in the firmware. In the parameter descriptor table (0x3c071f54, 32 bytes per entry) word 0 = name, word 1 = label, word 2 = step count and word 3 = pointer to an array of pointers to the step name strings. All readable arrays are listed in analysis/step_names.txt (layer mode, layer alloc, LFO shapes, filter type, drive type, mod sources and destinations). Confirmed by reading the strings.

- FilterType (6 steps): 0 Low-Pass 24dB, 1 Low-Pass 12dB, 2 High-Pass 24dB, 3 High-Pass 12dB, 4 Drive, 5 Bypass. This matches the round 8 code reading (types 0 and 1 one structure, 2 and 3 another, 4 drive, anything else unfiltered). Emulator check: types 2 and 3 behave as high-pass and 2 is steeper than 3 (about 94 dB against 61 dB down at 130 Hz with cutoff 0.7). The low-pass spectra were too noisy to use, so the 24 dB against 12 dB slopes are not measured. The two "dB" figures are not slope values, just the attenuation at one frequency.
- LFO shapes (7): Sine, Triangle, Pulse, Saw, Saw Up, Random, S & H. Same order as the code (0 sine, 1 triangle, 2 square, 3 falling saw, 4 rising saw, 5 white noise, 6 sample and hold); the match is by order, not traced.
- Mod destination names match the engine ids one to one: 0 Off, 1 Wave 1, 2 Wave 2, 3 Pitch (both oscillators), 4 O1 Pch, 5 O2 Pch, 6 O1 Vol, 7 O2 Vol, 8 Dirt, 9 Cutoff, 10 Resonance, 11 Vol, 12 Pan, 13 LFO 1 Speed, 14 LFO 2 Speed, 15 to 18 Wave Env A D S R, 19 to 22 Filt Env, 23 to 26 Amp Env. This closes the round 9 open point about ids 3 and 12.
- Mod source id 29 is Keytrack (the round 9 note had no name for it). Other source names match round 9.
- DriveType (5 steps): PNP, Tube, PickUp, Diode, Crunch, now read from the descriptor in step order (the round 8 note said the order was unconfirmed).

Layer allocation (engine note-on dispatcher fcn.42014cec; LayerMode at engine+0x3324, LayerAlloc at +0x3328): measured in the emulator in Multi mode with layers given different pitches.
- Layered (0): every active layer plays each note. Confirmed.
- Round Robin (1): notes alternate between the active layers only (A,B gave A B A B; A,C gave C A C A). No note was lost. My earlier reading of the decompile, that a note aimed at an inactive layer is dropped, was wrong or incomplete.
- Random Robin (2): all 12 test notes sounded, but 11 went to layer A and 1 to B in a two-layer test. That is far from an even split, so the selection rule is not understood (the code adds a random number to the last layer index; the random source fcn.4203fe88 was not checked). Unresolved.
- Midi Split (3): the note goes to the layer equal to the incoming MIDI channel (channels 1 to 4 = layers A to D), only if that layer is active; notes on channels 3 and 4 were silent with only A and B active. Confirmed.
- Solo A to D and Multi: as in round 15.

Not open any more: wavetable step to table mapping. Round 2 already has it (step s gives table s below 32, else s + 0x18, confirmed at three code sites).

Still open in the sound engine: LFO skew source (+0x10) and the retrigger flag (+6) setter; whether the filter coefficient g is tan-based; the meaning of per-part value k=0x35 and the update-or-reuse coefficient flag; the voice steal priority; the Random Robin rule; the low-pass slopes.

### Round 21b: follow-ups (layer allocation, LFO, filter, voice slots)

Layer allocation, corrected from the disassembly (the decompile of fcn.42014cec hid a 4-step search loop):
- Round Robin: next slot = (last + 1) mod 4, then the first ACTIVE layer from there (up to 4 tries); the chosen layer index is stored at engine+0x32e8 + ... (the "last" byte). Matches the test (A B A B, A C A C, no notes lost).
- Random Robin: slot = (last + rand()) mod 4, then the first active layer AFTER that slot. This is biased toward a layer that follows inactive layers: with A and B active, A is chosen for 3 of 4 random slots and B for 1 of 4; with A and C active it is 50/50. Matches the test (11 of 12 notes on A). rand() is newlib's (fcn.4203fe88, state seeded 0xabcd330e).
- This replaces the "Random Robin unresolved" line in Round 21.

LFO (fcn.42016648 and the voice start code):
- Retrigger flag (+6): set to 1 at the start of every voice for both per-voice LFOs (voice+0x566 and +0x582), together with held sample, phase and previous phase cleared (voice start code near ALL.c line 3585). The tick then holds the phase for one tick (phase step is multiplied by 0) and clears the flag. So a per-voice LFO (Sync = 0) restarts at every note; for shape S & H the flag forces a new random value. The shared LFOs (Sync above 0) get no such flag in the code I read, so they run free. Closes that open point.
- Skew (+0x10, 0..1, scaled by 65535, minimum 1): warps the phase so that the first part of the cycle (up to the skew point) maps to the first half of the waveform and the rest to the second half. 0.5 is a symmetric wave. I found no code that writes it after construction (searched the voice and engine decompile for the field offsets), so where it is initialised to a useful value is unknown. Emulator check: a sine LFO on pitch moved the pitch both above and below the base note (about 120 to 147 Hz around 131 Hz, tracked roughly), so the output is bipolar and not degenerate. Unresolved: the writer of the skew value.

Filter measurements in the emulator (half-octave band powers against Bypass; the noise in the reference level is about 4 dB):
- High-pass types: slopes about 22 dB per octave for type 2 (24 dB) and 12 dB per octave for type 3 (12 dB). Confirmed. At cutoff 0.7 the 4-pole type is still about 8 dB down at 5 kHz and 4 dB down at 7.6 kHz, so the effective corner is clearly higher than the 2.8 kHz that g = tan(pi fc / 44100) with the round 8 constants predicts for a plain 4-pole cascade. So the "g is tan-based" assumption is not supported; frequency mapping stays open.
- Low-pass types 0 and 1: in the emulator the spectrum did not change with cutoff at all (0.05 to 0.9, resonance 0), while the high-pass types did. This is unexplained. It could be an emulator problem with an instruction the low-pass code uses, or a mistake in my test; I do not believe the real unit has no low-pass filter. Do not use the emulator to judge low-pass filter sound until this is understood. Unresolved.
- Resonance 0.6 raises a peak near the cutoff in the high-pass types (about +16 dB at 0.5), as expected.

Voice slots (note-on to a layer, fcn.4037a4b0; read from the decompile only, so lower confidence than the items above): a layer holds 8 voice slots of 0x70 bytes starting at layer+0x10 (state byte at +0x11, 0 = free). The last used slot index is at layer+0x390. A new note starts at slot (last + 1) mod 8 and uses it directly if it is free and the number of sounding voices is below the layer polyphony limit (layer+0x392; 8 in a solo layer, 8 divided by the active layer count in Multi). Otherwise the code picks a slot to steal by state (values 1, 2 and 3 are handled with different priority). A mono/legato flag at layer+0x397 sends the note to the current slot instead. The exact steal priority was not worked out.

Still open in the sound engine: why the low-pass types do not filter in the emulator, the writer of the LFO skew value, the filter's per-part value k=0x35 and the update-or-reuse coefficient flag, the exact voice steal priority, the filter cutoff to frequency mapping.

### Round 21c: correction on the low-pass filter test

The Round 21b statement that the low-pass types showed no filtering is WRONG as a general claim. The first test used the default wavetable position, whose waveform has almost no energy above a few hundred hertz, so there was nothing for a low-pass to remove and the comparison measured a noise floor. Redone with wave position 63 (a sawtooth, from the generic table) and band powers against Bypass:
- Type 1 (Low-Pass 12dB) works: at cutoff 0.3 the level falls about 11 dB per octave from 1 kHz upward (-20, -31, -42, -52 dB at 0.95, 1.9, 3.8, 7.6 kHz). At cutoff 0.5 the high bands are down about 15 dB at 4.8 kHz, at 0.7 about 5 dB.
- Type 0 (Low-Pass 24dB) is inconsistent: at cutoff 0.5 the highs fall (about -15 dB at 4.8 kHz, 80 dB peak range), but at 0.3 the spectrum is flat and has an extra strong low component (the strongest peak sits at a different frequency than in the Bypass recording). I do not know whether this is the firmware, the emulator, or my analysis (the fundamental frequency detection was unreliable: the peak picker returned 66, 100 and 131 Hz for what should be the same note). Unresolved; test it again with a proper pitch tracker before drawing conclusions.
- The coefficient values in the live voice (read through the gdbstub at voice+0x410, +0x414) rise with cutoff as the formula says (type 1: 0.0185, 0.0577, 0.2267 at cutoff 0.3, 0.5, 0.7; type 0 about 1.7 times larger at the same cutoff). So the coefficient code runs in the emulator.
- Voice objects (layer A, 8 voices, stride 0x828, first at 0x3fcaaf78 in this build) can be read from the emulator by gdbstub: +0x4ac filter type, +0x4b0 ramped cutoff, +0x518 amp level, +0x530 amp stage. This is a handy way to see what the engine really does without audio.
- High-pass types are unchanged by this: 24 dB per octave for type 2 and 12 for type 3 (confirmed again with the sawtooth).

Open: the LP24 behaviour above; the cutoff to frequency mapping (the measured corners are higher than the plain tan(pi f / fs) estimate for both families).


## Round 22: effects pass (corrections, measurements, open points)
Sources: disassembly (read with scripts/r2d.sh, not only the decompile), the descriptor table, and one emulator measurement. No hardware test.

Step names (new file analysis/step_names_fx.txt). Only two FX parameters have named steps in the descriptor table: Tremolo Shape (Sine, Triangle, Square) and Drive Type (PNP, Tube, PickUp, Diode, Crunch). The other stepped ones (Chorus Shape 2, Chorus Stages 4, Flanger Shape 2, Phaser Shape 2 / Stages 5 / Model 2, Compressor AutoGain 2, Delay Link 2 / Sync 2 / RatioL 16 / RatioR 16, FX1Type and FX2Type 10) have no name array there; their labels come from display code (round 17 list: Triangle/Sine, Stages 2 4 6 8, Phaser Stages 2 4 8 12 16, Model Nave/PPG).

Corrections to round 11 (parameter locations)
- Chorus: Center (id 7) is the ramp at fx+0xc4 and Depth (id 4) is the ramp at fx+0xd4. Round 11 had Depth at 0xc4 and no Center. Confirmed from the process function: the ramp at object offset 0x24 (fx+0xc4) goes through c^2 x 44100 x 0.2 (the centre delay) and the one at 0x34 (fx+0xd4) through d^3 x 882 (the depth).
- Delay: DryWet (id 59) is at fx+0x710 (object +0x2c; used as the 2m - m^2 crossfade with a mod input added first). Full delay map: 59 DryWet 0x710, 60 Left ramp 0x6f0, 61 Right ramp 0x700, 62 Link 0x728, 63 Sync 0x72c, 64 RatioL 0x730, 65 RatioR 0x734, 66 Feedback 0x714, 67 HighCut 0x71c, 68 LowCut 0x720. The feedback value is clamped to 0..1 and then squared before use. Object +0x34 (fx+0x718) is read in the filter section as (1 - x)^4 but nothing in the setter writes it; probably a constant or a mod input (unconfirmed).
- Tremolo: the round 11 and round 17 notes were wrong in two places. In the setter, id 30 (Depth) uses the ramp at fx+0x144c/0x1450 and id 31 (Speed) is the plain value at fx+0x145c. The expression 0.1 + 39.9 x^2 that round 17 called the Smooth coefficient is the Speed mapping.

Measured (emulator, FX1 = Tremolo, sine, depth 1.0, smooth 0, sustained sawtooth, AM rate from the FFT of the output envelope)
| Speed | 0.1 | 0.2 | 0.3 | 0.4 | 0.5 | 0.6 | 0.8 | 0.9 |
| measured Hz | 0.49 | 1.65 | 3.61 | 6.3 | 9.9 | 14.2 | 25.4 | 30 (limit of my analysis band) |
| 0.1 + 39.9 x^2 | 0.50 | 1.70 | 3.69 | 6.48 | 10.1 | 14.5 | 25.6 | 32.4 |
So Tremolo Speed = 0.1 + 39.9 x^2 Hz (0.1 to 40 Hz), matches within about 3 percent. Smooth: its mapping is not decoded yet.
Tempo sync in the tremolo: the process function contains a tempo-synced path (rate = entry of the 32-entry table at 0x3c071960, index from the float at object +0x78 (default 23.0, so 0.5), multiplied by a double in the transport record), but it is only taken when the float at object +0x74 is 1.0 or more, the constructor sets it to 0.0, and I found no writer. The tremolo has no Sync parameter in the 75-entry table. So the table is dormant code, not reachable from the panel as far as I can tell (a writer through a computed pointer is not ruled out). This could be a cheap future feature (add a Tremolo Sync parameter).

FX1 and FX2 set to the same type: settled. In the FX processor (fcn.4200e700, instruction at 0x4200e964) the second slot is skipped when its object pointer equals the first slot's pointer (`beq a10, a12`). So the same effect twice simply runs once; there is no shared-state problem, but the second slot does nothing. The Bus 1 signal is still added before that test, so it is not lost. Emulator check: Tremolo in both slots (depth 0.5, speed 0.4) gave the same modulation depth (min/max ratio 0.26 both times) and the same rate (6.3 and 6.4 Hz) as Tremolo in one slot; a second pass would have deepened the dips.

Compressor AutoGain (kernel fcn.4202a008): when AutoGain is on, make-up gain = 0.8 x dB_to_linear(T / R - T), where T is the threshold in dB (0 to -50, negative) and R the ratio (1 to 100); cached and recomputed when a parameter changes (flag at kernel +0x35). The 0.8 is a constant in the constructor (object +0x88). Final gain = (compression gain) x (make-up) x (OutGain). That is: the amount of gain reduction that a full-scale signal would get, minus about 2 dB. The control function fcn.4200fb78 (vtable +0x10) does the per-block ramps and calls the kernel four times, once per group of 4 samples.

Phaser (partial, from the decompile of fcn.42012b18; the decompile merges two functions, so treat as provisional)
- Model value 1 (labelled PPG): a second-order all-pass section per channel. Coefficient a comes from the LFO through a lookup (fcn.42012ac4 on the table at 0x3fca541c); with p = the Feedback-slot value (0..1 after mod): c1 = (2 - a^2) - (2 - a) p a, c2 = (2 - a) p a - 1; the section uses (c1, c2, c2, c1, -1).
- Model value 0 (Nave): coefficient from the other table (0x3fca5014), smoothed with 0.9 / 0.1 one-pole filtering, then turned into an all-pass with fcn.4202a2b0 using a width of 0.008 + 0.1 p^2 (times pi). The Model enum labels are Nave and PPG; which value is which label is by order and not confirmed.
- The stage count maps through a table at 0x3c0700a8 (iVar3). The cascade over stages was not followed, so the all-pass count per stage setting is still unknown.

Still open after this pass
- Phaser: stage cascade and exact Model difference. Flanger and chorus feedback/filter details. Reverb topology versus a known design. Tremolo Smooth mapping and PhaseDiff. The role of delay object +0x34. Exact limiter behaviour was read but not measured. No hardware measurement of any effect.
- Not affecting the four planned improvements: effects are independent of the arp latch, envelope knobs, wavetables and time display, except that the firmware's ms/seconds formatters are shared with compressor and delay (see round 11).

### Round 22b: phaser follow-up
- Stage setting to section count: the table at 0x3c0700a8 holds 1, 2, 4, 6, 8 (read directly). The Stages labels are 2, 4, 8, 12, 16, so the label is twice the table value. The coefficient set per section has five values (c1, c2, c2, c1, -1), which is the shape of a second-order all-pass, so each table unit is most likely one second-order section (two "stages"). Interpretation, not proven. The object has room for 8 sections per channel (coefficient copies at stride 0x28 from +0x44, and +0x2c4 for the right channel).
- The decompile label 0x42012eb4 is not a second function; it is the second half of the process function 0x42012b18 (vtable +0x10 at 0x3c0070a4).
- Emulator measurement attempted (noise source through the phaser, 5 stage settings x 2 models, plus feedback 0.8): inconclusive. The Dirt noise is not white after the voice path (large bumps near 400 Hz and a falling top end), so notch counts could not be separated from the source spectrum. Observed only: Model 1 (PPG label) left the overall level higher and the spectrum flatter than Model 0 at the same settings, which is compatible with weaker notches, but this is not a clean result. A proper test needs a flat-spectrum input (an impulse or a swept sine through the Wave 1 path) instead of the Dirt noise.

### Round 22c: phaser impulse test and a correction to the master stage
Method (valid for hardware, unlike the audio-recording attempts). The emulator patches only touch ADC, SPI, flash, I2S descriptors and the panel input (8 addresses, none in the effects or voice code), so the effect code that runs is the unmodified firmware. To avoid the emulator's audio device (which drops samples when the guest is paused), the test stops the firmware at the call into effect slot 1 (0x4200e73d), writes a single impulse (0.25 in sample 0) into the input bus, and then reads the effect's own 16-sample float output buffer from firmware memory after every block (breakpoint at 0x4200e740, single-step off it between blocks, input bus cleared after the first block because the engine does not clear bus 0 when idle). 640 blocks = 10,240 samples per setting. Remaining emulator dependence: the QEMU floating point unit versus the ESP32-S3 FPU (both IEEE single precision; differences would be at the 1e-7 level). Scripts: scripts/emu_tests/cap.py, capset3.py, capan.py. Check run: bypass returned exactly the input (0.25, then zeros).
Lesson: the first attempts recorded the I2S wav while pausing the guest; that gave dropped samples and a constant Nyquist tone and must not be used for DSP conclusions.

Phaser results (LFO Depth 0 and Speed 0 so the filter is static; Center 0.5 unless stated; Dry/Wet 1.0)
- Stages: the label is the real stage count. Total phase rotation between 30 Hz and 20 kHz is about 177 degrees per labelled stage in both models: 2 stages 354 / 347 degrees (Nave / PPG), 4 stages 706 / 696, 8 stages 1408 / 1391, 12 stages 2112 / 2088, 16 stages 2815 / 2786. Each stage is a first-order all-pass equivalent (180 degrees DC to Nyquist). The table 1, 2, 4, 6, 8 at 0x3c0700a8 is therefore the number of second-order sections (two stages each).
- Magnitude with Dry/Wet 1.0 is flat within about +-2 dB in both models (not an exact all-pass, small ripple).
- Dry/Wet law confirmed numerically: wet = 2m - m^2, dry = 2(1-m) - (1-m)^2. At m = 0.5 both are 0.75, so for a signal in phase the sum is 1.5 (+3.5 dB peak); measured first sample 1.3634 = 0.75 + 0.75 x 0.8179 for Nave 8 stages, and 1.5 for PPG. m = 0 is an exact pass-through (1.0) and m = 1 is wet only.
- Center sets the frequency of the stages: with Spread 0.7 the frequency where half of the phase rotation has happened is 1.3 kHz (Center 0.2), 1.35 kHz (0.5), 4.4 kHz (0.8) in Nave, and 5.1 kHz at 0.8 in PPG. With Spread 0 and Center 0.5 it is 504 Hz (Nave), which matches the table value for index 0.5, 466 Hz, within the measurement resolution.
- Spread: in Nave it staggers the stage frequencies upward (half-rotation point 504 Hz at 0, 573 Hz at 0.3, 1348 Hz at 0.7, 2356 Hz at 1.0). In PPG it sets the bandwidth of each section and at Spread 0 the sections cancel to a plain polarity inversion: the response is -1 for an odd number of sections and +1 for an even number (measured first sample -1.0000 for 2 stages and +1.0000 for 4+, phase swing 3 degrees). So PPG with Spread 0 does nothing audible except polarity.
- Feedback is bipolar: loop gain = 2f - 1. In PPG with Spread 0 the impulse response was exactly 1, r, r^2... with r = 0.2 at f = 0.6, -0.4 at 0.3, 0 at 0.5 and 0.8 at 0.9, which confirms the mapping.
- Stability: at Feedback exactly 0.0 or 1.0 (loop gain -1 or +1) the response never decays (still at the full 0.25 amplitude after 10,240 samples) in both models; at 0.9 the response rings for a long time with peaks of +14 dB (Nave) and +30 dB (PPG). Values between about 0.1 and 0.8 decay normally. So Feedback at the end stops is a sustained oscillation. This is a property of the firmware math as measured; I did not check what it sounds like on hardware or whether the limiter hides it.
- Measured coefficient tables at boot: the Nave table at 0x3fca5014 holds normalised frequency f/fs, exponentially spaced from 11.6 Hz (index 0) to 18.3 kHz (index 255), e.g. 466 Hz at index 128; the PPG table at 0x3fca541c holds 2 sin(pi f / fs) of the same frequencies (1.9275 at index 255). Both are built by the phaser constructor loop at boot.
- Not decoded: how Model PPG uses the stage-pair structure internally (the section coefficients are (c1, c2, c2, c1, -1) and with Spread 0 they collapse as above), exact Nave stage frequencies.

Correction: the master filter after the effects is NOT a pass-through. Round 11 and 17 said the biquad at fx+0x60 is initialised as a pass-through and nothing else writes it. In the running firmware the five coefficients are 1.9532, -0.95423, 1.0071, -1.9528, 0.94755 (unchanged by the Flavour value; tested with Flavour 0, 0.5, 1.0, 0.17). They are written at FX initialisation by `fcn.4202a308(fx + 0x5c, 0.0052721, 4.56)`, the same low-shelf routine the EQ uses, i.e. a fixed low shelf of +4.56 dB below roughly 37 Hz (frequency argument read as 2 pi f / fs; that reading is an inference from the number, 37.0 Hz). Cross-check: the measured bypass impulse response of the whole output chain has a slow positive tail whose total gives a DC gain of about 1.7 (+4.6 dB). It applies to every sound regardless of settings. Flavour is still without a found consumer.

### Round 22d: reverb impulse test (same method as round 22c)
Impulse 0.25 into effect slot 1, effect's own float output read from firmware memory (so no emulator audio path). Mod Depth set to 0 so the reverb is static; Time 0.75, Color 0.5, Pre-Delay 0, Dry/Wet 1.0 unless stated. Captures up to 3 s (132,288 samples). Script: scripts/emu_tests/rv_run.py (uses cap.py).
- Stereo: left and right differ (different tap sets). With Pre-Delay 0 the first output is at sample 430 on the left (9.8 ms) and 574 on the right (13.0 ms). Left early taps (sample: value) 430: -0.038, 832: -0.037, 980: -0.037, 1382: -0.037, 2965: +0.044, 3563: +0.043. Peak amplitude 0.044 (left) and 0.050 (right) for the 0.25 impulse.
- Decay time (fit of the energy envelope, 50 ms bins): Time 0.25 about 1.1 s to fall 60 dB, 0.5 about 2.1 s, 0.75 about 4.7 s. Time 1.0 does not decay at all over the 2.5 s captured (slope 0.0 dB/s), so the top of the knob is an endless reverb; how it behaves over minutes (stable or slowly growing) is not measured. Slopes in dB/s: -52, -28, -13, 0.
- Dry/Wet is NOT the 2m - m^2 law used by chorus, phaser and delay. Measured: wet amplitude = m^2 (0.062, 0.247, 0.549, 0.808, 0.895, 0.979 at m = 0.25, 0.5, 0.75, 0.9, 0.95, 0.99; squares are 0.0625, 0.25, 0.5625, 0.81, 0.9025, 0.98). The dry signal stays at full level (1.000) for m up to 0.9 and then falls steeply: 0.975 at 0.95, 0.199 at 0.99, 0 at 1.0, which fits dry = min(1, 20 x (1 - m)). So below about 0.9 the reverb is purely additive: turning Dry/Wet up never lowers the dry sound. This explains why the default 0.25 is a subtle amount. (Fitted from six points, not read from code.)
- Color is a spectral tilt on the whole response. Relative to 1 kHz: Color 0.0: +7 dB at 100 Hz, -5 dB at 2 kHz, -19 dB at 8 kHz, -29 dB at 16 kHz. Color 0.5: flat within 0.2 dB. Color 1.0: -12 dB at 100 Hz, +6 dB at 2 kHz, +16 dB at 8 kHz, +17 dB at 16 kHz.
- Pre-Delay (shift of the first strong tap relative to Pre-Delay 0): 0.05 gives 864 samples, 0.15 gives 2864, 0.3 gives 6432, 0.5 gives 12400, 0.75 gives 21760, 1.0 gives 33136. A fit gives about 16,500 x (x + x^2) samples (residuals within 25 samples, i.e. the 16-sample rounding). This is slightly larger than the 16,370 (x + x^2) from the code reading in round 17; the true constant is probably not exactly what I read, so treat the measured table as the reference. Maximum about 0.75 s (33.1k samples) at Pre-Delay 1.0, so a clamp near 32.7k plus the 400-sample base latency is plausible.
- Not measured: Mod Depth and Mod Rate effects. Not identified: the topology (the plate-style guess in round 17 is still unverified); the early tap pattern above would be the starting point.

## Round 23: static mapping without the emulator
MIDI clock-enable byte (open since round 12): resolved. The byte at +0x220 that the clock handler `fcn.4201beac` tests is set to 1 by the initialiser `fcn.4201a90c` (store at 0x4201a94f: `addmi a11, a7, 0x200; s8i a12(=1), a11, 32`). That initialiser also clears the 64-entry timestamp ring at +8 (128 words), zeroes the fields at +0x208, +0x210..+0x250 and sets the tempo double to the default; the layout matches the handler exactly (ring +8, head +0x208, state +0x218, count +0x21c). I searched the whole text segment (200,000 instructions) for stores to offset 0x220: apart from this initialiser the only other hits are in the Globals object (fcn.42029100/120/170/1e4, a different object) and one engine-object initialiser at 0x42016f65. Nothing clears the byte. So MIDI Start, Stop, Continue, Clock and Song Position are always processed; the only gate on external sync is the Sync Source setting inside the tempo routine `fcn.4201c074` (Intern gives a fixed 120 BPM). For improvement 1 (arp/sequencer latch) this means incoming MIDI transport does not need any enable step. Static analysis only, nothing tested.

## Round 24: parameter value display path (static, no emulator)

- Generic value-to-text function: fcn.42019024(paramId, buf=a4, size=a5, value=float a3). Param id < 0x150 maps through id % 0x54 into the per-layer table at 0x3fc9fb78 (0x1c bytes per entry); id 0x150.. uses table 0x3fc9e2dc (FX). Entry +6 = descriptor index (via fcn.42018a50, FX table 0x3c06f5c4 for FX, sound table 0x3c071f54), entry +8 / desc +8 = step count.
- Logic: (1) if descriptor has formatter (desc word 7, offset 0x1c) non-null: call formatter(buf, size, value) (callx8 at 0x42019082, a10=buf, a11=size, a12=value float). (2) else if steps > 0: index = clamp(trunc(value + 1e-6), 0, steps-1), print name from step-name array via "%s". (3) else if bipolar flag (desc +0x18 / word 6 nonzero): print (value-0.5)*2 with "%+.2f". (4) else print value with "%.2f" (fallthrough at 0x42019134).
- Envelope A/D/S/R descriptors (WaveEnv 44-47, AmpEnv 52-55, FiltEnv 57-60) have no formatter and no steps, so they show raw 0..1 as "%.2f" (A/D/R) and the same for Sustain.
- Example formatter 0x42017758 (AmpEnvVelocity): entry a1,48; value*100 -> double -> snprintf-style fcn.4204090c(buf,size,"%.0f%%",d). Signature to copy for a new formatter: (buf, size, float value), return nothing.
- Callers of fcn.42019024: 0x4201ca.. (knob/value draw, also computes bar fill from value*120), 0x4201f062 (periodic value readout, with 500 ms hold time check), 0x42060.. area (several UI pages). One function covers all knob popups, so a single descriptor pointer change affects every display of that parameter.

Improvement 4 plan (hardware-valid, small): add a new formatter function (value -> time in ms or s using the already-derived envelope time law, 4.605 x tau for 99 percent) in free flash, then patch word 7 of the 9 envelope descriptors (A/D/R only; Sustain stays percent via 0x42017758). Descriptor addresses = 0x3c071f54 + idx*32 + 0x1c. No change to DSP code needed. Still to check: free space for the new function in the app image, and whether descriptors sit in a read-only segment covered by an image checksum/hash (rebuilt image needs its SHA/checksum recomputed anyway).

Decision (user): envelope time display shows time to 99 percent (envelope_times.csv column time_to_99pct_ms), ms below 1 s, seconds above (e.g. "250ms", "1.2s"). Unconfirmed on hardware, plus or minus 15 percent.

## Round 24b: where to put the new envelope formatter (open)
- No zero/0xFF runs of 64 bytes or more in the IROM segment (0x42000020..0x42053398). DROM has some but it is not executable.
- Static dead-function search (functions.json calls plus 4-byte address literals) is unreliable: it flags functions known to be used (for example fcn.4200d0d8), so it cannot prove a function is safe to overwrite. Not used.
- Options: (a) append about 200 bytes to the IROM segment end (needs image rebuild: segment length, 64 KB MMU alignment of following data, checksum, SHA-256; partition size unknown, the app is 581 KB and the emulation slot is a guess of 1 MB); (b) overwrite a function proven unused by a gdb watchpoint/breakpoint run in the emulator (still not proof for hardware); (c) append to the iram1 segment end (risk: IRAM heap may use that space).
- The original UF2 and the real partition table are not in the workspace, so the real app slot size cannot be checked here.

## Round 24c: original UF2 inspected (user attached it)
- Saved as ref/PROTEIN_v102_original.UF2 (1,189,376 bytes, 2323 blocks of 256 payload bytes, family 0x12e6002f, flags 0x2000, target address starts at 0 and runs contiguously to 0x91200 = offset inside the app slot, not a flash address).
- Payload = 594,688 byte ESP app image (0x91300): magic 0xE9, 6 segments, entry 0x40379204, flash 0x3f (8 MB?), hash-appended flag set. Segment file offsets: DROM 0x20, DRAM 0x22f38, IRAM 0x28594, IROM 0x30020 (0x42000020, 64 KB alignment OK), IRAM1 0x833a0, RTC 0x9122c. Segment table ends 0x9124c, XOR checksum byte at 0x9124f (0x8f, verified), SHA-256 at 0x91250 (verified, matches). After that 144 zero bytes (UF2 padding to the block size).
- IROM segment bytes in the UF2 equal the ELF. So a rebuilt image is feasible with a script: grow the IROM segment, shift later segments, recompute XOR checksum and SHA-256, rewrite UF2 blocks (block numbers/total, addresses, 256-byte payloads).
- Still unknown: app slot size in the real partition table, and whether the bootloader checks image size beyond the SHA. Growth needed is about 200 bytes (the file grows by one or two UF2 blocks).

## Round 22e: chorus and flanger impulse results (emulator capture of FX output buffers, 44.1 kHz assumed from engine init)
Method as phaser/reverb (impulse 0.25 in at FX1 input, read FX1 outL/outR floats). Peak positions in samples.

Chorus (obj 0x3fca3b9c; base: Speed 0 so no modulation):
- Center (chce): first tap delay = about 8900 x ce^2 samples (+ about 2): ce 0.06 -> 34, 0.3 -> 796, 0.5 -> 2239, 1.0 -> 8918 (202 ms). Ratios 2.78 and 3.98 match squares.
- Spread (chsd): spacing between voice taps = about 882 x sd samples (20 ms max): sd 0.4 -> 353, 0.75 -> 661, 1.0 -> 882; sd 0 puts all taps on one position (peak sums, 1.35).
- Stages (chst) 0..3 = 1..4 taps (st0 one tap, st1 two, st2 three, st3 four), amplitude per tap falls with count (0.78, 0.68, 0.58, 0.48 on L).
- Feedback (chfe) is bipolar around 0.5: loop gain about 1.5 x (fe - 0.5) (fe 0 -> -0.75, 0.25 -> -0.38, 0.5 -> 0, 0.75 -> +0.38, 1 -> +0.75). Max |0.75|, always stable. Repeat period at ce 0.3 is about 1505 samples (about 1.9 x the tap delay; loop contains more than the single tap: not resolved).
- Dry/Wet (chdw): wet level relative to dw=1 at 0.25/0.5/0.75 = 0.47/0.78/0.95. Close to 1 - (1-dw)^2 (0.44/0.75/0.94) which is the exact law for the flanger, but not identical here (dry path interferes). Unresolved.
- High-Cut (chhi) = 1 gives exactly zero output; Low-Cut (chlo) = 1 gives about 0.007 peak (nearly silent); 0.5 reduces and changes the shape. Polarity/frequency mapping not resolved (needs spectrum at several settings).
- L and R differ slightly (R about 3 percent higher) with Speed 0: stereo spread of the taps, not analysed.

Flanger (obj 0x3fca4488; base: Speed 0):
- Center (flce): delay = 1 + 1103.5 x ce^2 samples (25 ms max): ce 0 -> 1, 0.1 -> 12, 0.5 -> 277, 1.0 -> 1103.5 (half-sample interpolation). Same square law as chorus.
- Feedback (flfe): bipolar around 0.5, zero at 0.5. Loop gain g = sign(x) x 0.976 x (1 - (1-|x|)^3) with x = 2 x (fe - 0.5): fe 0.25 / 0.75 -> -/+0.853, fe 0 / 1 -> -/+0.976. Negative side alternates sign each loop. Stable (max 0.976).
- Dry/Wet (fldw): wet gain = 1 - (1-dw)^2 (0.4375 / 0.75 / 0.9375 / 1 at dw 0.25 / 0.5 / 0.75 / 1; measured 0.438 / 0.75 / 0.9375).
- PhaseDiff (flpd): no effect with Speed 0 and mono input, so its meaning is unresolved (likely LFO phase offset between L and R). Needs a test with Speed above 0.
- Loop delay equals the flanger delay (12 samples at ce 0.1, echo period 12).

## Round 22f: flanger PhaseDiff, Depth, Speed (44 impulse captures, 100 blocks each, flce 0.5 = 277 samples static delay)
- PhaseDiff (flpd) is the LFO phase offset of the right channel against the left, as a fraction of one LFO cycle. pd 0 and pd 1 give identical L and R (same delay on both), pd 0.25 / 0.5 / 0.75 give different delays L vs R (for example dp 0.5, sp 0: L 323 / R 363 at pd 0.25; L 326 / R 282 at pd 0.75). With Depth 0 (fldp 0) pd has no effect and both channels sit at the static 277 samples.
- Depth (fldp) modulates the delay: dp 0.5 gave 281 to 376 samples, dp 1.0 gave up to about 1555 samples, more than the static maximum of 1103.5, so the LFO offset is added to Center before the square law (delay about 1103.5 x (ce + lfo x depth)^2, not clamped to 1.0; exact scaling unresolved). At dp 1.0 and Speed 0 the left output was zero in all captures (delay beyond the 100-block window, or a negative/clipped delay: unresolved).
- Speed 0 (flsp 0) does not freeze the LFO: two identical settings loaded at different times gave different delays (321 vs 338, 328 vs 360). The lowest Speed value is a slow running LFO, not a stop.
- Shape (flsh): 0.5 and 1.0 gave different delays on L and R (and R was zero at 1.0), a single snapshot each, so the waveform meaning is unresolved; needs several captures per value over time.
- Limitation: one impulse is one LFO snapshot with an unknown start phase, so these captures show channel relation (pd) but not the waveform or the rate. A rate and waveform measurement needs repeated impulses over time (like the tremolo test).

## Round 22g: flanger LFO rate, depth, waveform (repeated-impulse method)
Method (hardware-valid, reads FX1 output floats): impulse injected at the FX1 call every 24 blocks (384 samples = 8.7 ms), peak position of the response tracked per impulse. Needed two breakpoints (before the call to inject, after it to read), because the engine clears bus 0 every block and the output buffers are reused. Single-step then continue can land in an interrupt and re-hit the same breakpoint, so every stop is checked by reading the PC (scripts /tmp/harness/cap2.py, fx_run2.py; copy to scripts/emu_tests/). Static delay was ce 0.2 = 45.1 samples.
- Waveform: Shape (flsh, 2 steps) 0 = triangle (constant slope), 1 = sine (peak slope 1.6 x triangle = pi/2). Value 0.5 behaves as 0 (step index = trunc(v + 1e-6)).
- Rate: about 20 Hz x Speed^4 (measured Speed 0.5 -> 1.25 Hz, 0.75 -> 6.3 Hz, 1.0 -> 20.0 Hz, 0.25 -> about 0.076 Hz; Speed 0 is not stopped, about 0.005 Hz at most from a 10-window slope, floor unresolved). Fits 20 x 0.0625 = 1.25, 20 x 0.316 = 6.3.
- Depth: the LFO only adds delay above the static Center value (min equals static delay 45.1 samples). Peak sweep above static: Depth 0.15 -> 0.7 samples, 0.3 -> 11.4 samples; from the earlier ce 0.5 captures Depth 0.5 -> about 99, Depth 1.0 -> about 1280 to 1555 samples. Approximately 1400 x Depth^4 samples (about 32 ms at 44.1 kHz at Depth 1). Exponent about 4, not exact at the high end.
- PhaseDiff: right channel LFO phase offset as fraction of a cycle confirmed again (pd 0.5: L rises while R falls, same slope).
- Left/right peak amplitudes alternate with fractional delay (interpolation), not a level change.

## Round 22h: chorus High-Cut and Low-Cut (impulse spectra, wet path only, 44.1 kHz assumed, emulator)
Setup: chorus, 1 tap (chst 0), ce 0.06, dw 1, no modulation, reference chhi = chlo = 0. Spectrum = FFT of the 100-block impulse response divided by the reference.
- High-Cut (chhi): first-order low-pass, about 6 dB per octave. -3 dB point: 0.1 -> above 20 kHz (-1.1 dB at 20 kHz), 0.25 -> about 14 kHz, 0.5 -> about 5.1 kHz, 0.75 -> about 0.7 kHz, 0.9 -> about 150 Hz, 1.0 -> output exactly zero (wet path muted). Roughly exponential in the setting. At 0.9 the whole response is also lowered (-4 dB at 50 Hz), so the overall level falls as the corner falls; 3 dB points at 0.9 are rough.
- Low-Cut (chlo): first-order high-pass, about 6 dB per octave. -3 dB point: 0.1 -> about 50 Hz, 0.25 -> about 700 Hz, 0.5 -> about 4.4 kHz, 0.75 / 0.9 / 1.0 -> the corner estimate stalls near 6.6 to 6.9 kHz while the passband level keeps dropping (20 kHz gain -5.9 dB at 0.75, -13.7 dB at 0.9, -21 dB at 1.0): at the top of the range the filter is a very high corner, not a clean mute.
- Not measured: whether these filters sit in the feedback loop (they were measured with feedback 0.5 = no feedback), L/R differences, and the exact mapping formula (looks like corner frequency exponential in the setting).

## Round 25: Info menu and the UI page-state table (static)
- Global menu (labels at 0x3fca04d4: Info, Map CC, Mappings, Close; count 4 at ui+0x974, items ptr ui+0x978, select callback fcn.4201d274 at ui+0x988). Selection index: 0 Info -> fcn.4201d16c(ui, 9); 1 Map CC -> clears the info text ui+0x16bc, sets ui+0x17bc = ui+0x79c, flag ui+0x16b4 = 1 and deadline ui+0x16b8 = now_ms + 1,200,000 (20 minutes) so CC learn mode times out after 20 min; 2 Mappings -> fcn.4201d16c(ui, 10); 3 Close -> nothing (menu just closes).
- fcn.4201d16c(ui, state) switches the active page object (ui+0x24) and saves the previous state at ui+0x129c / new at ui+0x1298. State -> page object: 0 -> ui+0x2c, 1 -> ui+0xb8, 2 -> ui+0x198 (front panel), 3 -> ui+0x780, 4 -> ui+0x534, 5 -> ui+0x548, 6 -> ui+0x584, 7 -> ui+0x5ac (sound list), 8 -> ui+0x75c, 9 -> ui+0x790 (Info), 10 -> ui+0x79c (Mappings, vtable 0x3c073238, handler fcn.4201dda8), 11 -> ui+0x7ac; any other value clears the page. If the current page is ui+0x7ac only state 3 is accepted (it returns to ui+0x780).
- The Info page (ui+0x790, vtable 0x3c0738ec) draws only two things: the title "Protein" and the line "Version %d.%02d b%d" (draw method fcn.42026198; the other methods are stubs). It shows nothing else (no serial, no build date, no free memory).
- The version numbers are code constants: the draw call passes 1, 3, 3, so the screen would read "Version 1.03 b3". The MIDI sysex firmware-version reply (MidiSysexController__sendFirmwareVersion fcn.4201bab4) builds the same "%d.%02d b%d" string from constants 1 and 3. The build info string in the image header is different ("v1.02_b2-232-gdda56c3c-dirty", "Sep 21 2026"). Unconfirmed why they differ (possible: protocol/UI version separate from the release name). Not checked on hardware.
- Consequence for the planned mods: a modified build should change these constants (or the title string) so a modded unit is recognisable on the Info screen; the constants sit inside movi instructions (small immediates), so the patch is trivial.
- Resolved from the open list: page-object identities ui+0x534 (state 4), 0x548 (5), 0x584 (6), 0x75c (8), 0x780 (3), 0x790 (Info), 0x79c (Mappings), 0x7ac (11). What pages 4, 5, 6, 8, 3 and 11 are by name is still not resolved (only that they are state numbers).

## Round 25b: page identities by state (from strings in each page's methods, static; names are inferred, not seen on a unit)
| state | object | vtable | strings in its methods | reading |
|---|---|---|---|---|
| 4 | ui+0x534 | 0x3c073688 | "Play", "Note", "Octs", "%s%d" | Play/arp note page (play mode, note, octave range) |
| 5 | ui+0x548 | 0x3c0736d0 | "Sw %d", "Gt %d", "Record", "Edit", "Recording...", "%d steps", " Stop " | sequencer step record/edit page (switch and gate per step) |
| 6 | ui+0x584 | 0x3c073758 | name pointer "Sequencer" (ui+0x590), "[%d]", "%.0f%%" | sequencer step value page (percent values) |
| 7 | ui+0x5ac | 0x3c0737ac | "Layer %c", "Vol", "Active", "Off", "Actions", "%03d %s", " Select ", " Load " | sound list (already known) |
| 8 | ui+0x75c | 0x3c0737dc | "Multi Mode", "Single Mode" | multi/single mode choice page |
| 3 | ui+0x780 | 0x3c07380c | none; draw method fcn.4202415c shows one global setting: name text, value text (fcn.4202925c, value from the global array 0x3fccfd14 via fcn.4204f8d0) and a bar drawn from the setting's min..max (fcn.42029370) | global settings page, one setting at a time (inferred) |
| 11 | ui+0x7ac | 0x3c073830 | "0: %.1f < %.1f", "1: %.1f < %.1f" (fcn.420244c8) | debug/diagnostic value view; only reachable from state 3 (state change code at ui+0x7ac only accepts state 3 to leave) |
- State 9 Info and state 10 Mappings are in Round 25. Page 0x584 draw function fcn.42022d98 is the sequencer step readout noted earlier.
- Still unknown: how a user gets from state 3 to state 11 (button combo), and which control opens state 3 and 8.

## Round 25c: how each page is reached (button handler fcn.4201d518, static)
- Most of this was already in BUTTONS.md (round 5); this round confirms it from the dispatcher and adds the dead page. Button ids as in BUTTONS.md (5 = Shift, byte at ui+0 is the shift flag): 0-3 -> state 7 (sound list/layers), 4 -> state 8 (Multi/Single), 6 -> state 3 (settings list, 134 items at 0x3c0752a0), shift+6 -> popup menu flag ui+0x960 (Info / Map CC / Mappings / Close), 7 -> state 2 (front panel), shift+7 -> fcn.42021b0c on ui+0x2c, 8 -> state 4 (held; release returns to the previous state ui+0x129c), shift+8 -> state 5, 9 -> state 0, shift+9 -> state 1 after fcn.42023274 on ui+0xb8. State 6 comes only from page 5 "Edit" (call at ALL.c line 64745).
- Page ui+0x780 (state 3) is therefore the settings list; my Round 25b description matches (name, value text, bar from min..max).
- State 11 (ui+0x7ac, the "0: %.1f < %.1f" debug view, draw fcn.420244c8): no call site anywhere passes 11 to fcn.4201d16c, and fcn.420244c8 has no direct caller (only the vtable). The page cannot be reached in v1.02 as built: dead debug code (not verified on hardware).
- Previous-state variable (ui+0x129c) is only copied from the current state, so it cannot introduce state 11 either.

## Round 26: fcn.42016098 = LFO update (static)
- Called twice from the voice render (fcn.4037bfb0): for the own-LFO objects voice+0x560 (when voice+0x5a8 points at it) and voice+0x57c. Not the same function as the LFO tick fcn.42016648 from round 9 (that one was the per-block wrapper; this is the phase and waveform step).
- Object fields: +0 s16 raw output, +2 u16 phase, +4 u16 previous phase, +6 retrigger flag (cleared at the end), +8 float rate, +0xc float rate modulation scale, +0x10 float skew (0..1), +0x14 byte shape, +0x18 float output (-1..1).
- Phase step = clamp(rate x 127 x 64 + ticks x rate x scale x 4, 0, 0x1fff) x ticks elapsed, added to the 16-bit phase. A retrigger (+6 set) sets ticks to 0 for that step.
- Skew: break point = skew x 65535 (min 1). Phase below it is stretched to 0..32768, phase above it to 32768..65535, so skew moves the middle of every shape (0.5 = symmetrical).
- Shape byte: 0 sine (1024-entry table at 0x3c070540, linear interpolation), 1 triangle, 2 square, 3 falling ramp, 4 rising ramp, 5 noise (new random value every update), 6 sample and hold (new random value when the phase wraps or on retrigger). The mapping from the UI Shape value to this byte is not checked.
- Output = raw / 32768. The skew setter (who writes +0x10) is still not found.

Round 26 correction and skew result: fcn.42016098 duplicates what round 9 already recorded for fcn.42016648. They are two byte-for-byte copies of the same LFO update (482 and 486 bytes): 42016098 is called from the voice render for voice+0x560 and voice+0x57c, 42016648 is called from the tick (fcn.4037e71c area, ALL.c lines 6220 and 6224) for the global LFOs at engine+0x4030 and +0x404c. Skew (+0x10) result: no static store to it exists in the voice or global init code (global init at ALL.c line 50737 only zeroes +0, +4 and sets +6 = 1). Measured in the emulator at the first FX call: the skew of the global LFO1 and LFO2 objects (engine array 0x3fcbdcc4, stride 0x4074, +0x4040 and +0x405c) is 0.5 in all four parts, so something writes 0.5 outside the code I can see (bulk copy or init table). The voice-own LFO skew was not read. Effect: skew 0.5 = symmetrical, no phase warp.

## Round 27: reverb core structure (static, fcn.4202b018 block process, fcn.4202aa9c network, fcn.4202af24 output taps, fcn.4202ace0 gains, ctor fcn.4202ad50)
- One 65,536-float circular buffer (first 16 floats mirrored after the end), 16 samples per call. The block writes inL + inR (mono sum) at the write position, then a 3-tap smoothing filter [0.05, 0.90, 0.05] (ctor +0x3c8, +0x3c4, +0x3cc) on the sample at the Pre-Delay offset (obj+0x3c0). Everything after that happens in place in the buffer.
- Network (fcn.4202aa9c), in this order: (1) 8 in-place allpass sections (count 8 at obj+0x28, 16-byte records from obj+0x2c: two 16-bit lengths, one tap offset, two float gains; first three lengths 206, 365, 915 samples, later 1317, 1333, 2307 and others) = input diffusion. (2) 2 modulated delay taps (count obj+0x16c = 2, 0x24-byte records from obj+0x170): each has a triangle LFO whose rate is Mod Rate / 44100 x 16 per block and depth smoothed at 0.01 per block, writing a linear-interpolated read back into the buffer. (3) 2 three-tap filters (count obj+0x200 = 2, 16-byte records from obj+0x204) whose outer taps are scaled by the Time gain argument: damping inside the loop. (4) 2 recirculating delays (count obj+0x244 = 2, 12-byte records from obj+0x248, lengths 0x416e = 16,750 and others) that add the buffer sample from an earlier offset times the Time gain to the current one: the feedback of the tank.
- Output (fcn.4202af24): 7 taps summed into the left output (offsets from obj+0x280: 0x482a, 0x577d, 0x695e, 0x78e1, 0x1449, 0x234b, 0x3278) and 7 into the right (obj+0x320: 0xb02, 0x1d8e, 0x2931, 0x3b92, 0x529a, 0x606f, 0x6e42), each with a gain from fcn.4202ace0: gains are g, a x g, -a x g, a^2 x g, -a^3 x g ... where a = 1 for the Time-smoothed value above 0.5 (then all gains equal) and a = 5 - 4t scaled for t = 2 x value below 0.5. So the tap level pattern changes with Time below the middle of the knob.
- Smoothing: the two control values (obj+0x18 and +0x20 against targets +0x1c and +0x24) move 1 percent of the distance per 16-sample block (about 4 ms time constant at 44.1 kHz) so knob moves do not click. When a flush counter (obj+0x3f0) is above 0 (after a parameter change that needs it) the buffer is cleared in 16-sample steps and the output is passed through; for the smoothed Dry/Wet value above 0.45 the pass-through (dry) buffers in_a5 and in_a6 are multiplied by max(0, 10 - 20 x value), which is the dry fade seen in the round 22d measurement (dry stays at full level, then falls to 0); the exact value-to-Dry/Wet mapping was not re-derived.
- Comparison with a known design: this has the elements of a plate or Dattorro-like reverb (input diffusion allpasses, modulated tank delays, damping, 7 + 7 multi-tap output) but the structure is a single-buffer network with 8 diffusers, not Dattorro's 4 + 2 cross-coupled tanks, so it is not a copy of the published design. The exact feedback routing (which recirculating delay feeds which) is not traced yet; that needs the record contents at runtime.
- Mod Depth and Mod Rate: they drive the 2 modulated taps only (triangle LFO, no random component seen). Mod Rate = x^3 x 20 Hz as before; the depth scale 5 is applied to the per-record depth.

## Round 27b: reverb network decoded from the constructor records and the control (static), with a check against round 22d
Control fcn.42013420 (outer object O; core object C = O+0xc is what fcn.4202b018 sees):
- Dry/Wet m (O+0x404 + modulation, clamped 0..1): O+0x3dc = m^2; core target C+0x1c (O+0x28) = m^2 / 2, smoothed into C+0x18. The core then multiplies the dry pass-through buffers by max(0, 10 - 20 x C+0x18) once C+0x18 >= 0.45 (m >= 0.949). That gives dry = 10 - 10 m^2 above m = 0.949: 0.975 at m = 0.95 and 0.200 at m = 0.99, exactly the measured 0.975 and 0.199 in round 22d; wet gain m^2 matches too. So the round 22d Dry/Wet law is now confirmed from code (dry = min(1, 10 - 10 m^2), wet = m^2).
- Which field is which: O+0x400 = Time, +0x404 = Dry/Wet, +0x408 = Color, +0x40c = Pre-Delay, +0x410 = Mod Depth, +0x414 = Mod Rate (depth = x 5 for the first modulated tap; rate = x^3 x 20 for the second record). O+0x418 feeds C+0x14 = 2f and C+0x10 = 2 - 2f (a 7th control value; not one of the six parameters, source not identified).
- Time t (O+0x400 + modulation, clamped 0..1): stored in C+0x20 (target C+0x24, smoothed 1 percent per block). It is passed as the gain argument a5 of the network, so Time is directly the feedback gain of the recirculating delays and the outer taps of the damping filters. fcn.4202ace0 also sets the 7 + 7 output tap gains from it (same pattern both sides, in tap order): for t > 0.5 [g, g, -g, g, -g, -g, -g]; for t <= 0.5 with u = 2t: A = g x (5 - 4u), B = u x A, C = u x B and the pattern [A, B, -B, C, -C, -C, -C] (equal to g at t = 0.5, only the first tap loud at low t). g = C+0x18 = m^2 / 2.
- Color c (O+0x408), v = 2c - 1: three coefficients are written to C+0x3c4 (k), C+0x3c8 and C+0x3cc, and to every in-loop filter record (k at +0x214, then +0x218 and +0x21c). For v > 0: s = min(0.3 v^2, 1); k = 0.99 - 1.69 s; coefficients (k, (k + 1)/2, -(k + 1)/2). For v <= 0: k = -0.99 - 1.89 v (0.9 at c = 0); both outer coefficients (1 - k)/2. The filter is applied as buf[x+1] = outer1 x buf[x+1] + k x buf[x] + outer2 x buf[x+2] (so k weights the older neighbour). The constructor default is k = 0.9, outer 0.05 and 0.05. The tilt behaviour measured in round 22d comes from these coefficients; the exact frequency response was not derived here.
Network (head R = C+8; ages relative to R; the buffer is circular 65,536, ops run per block of 16; the input is written at W = C+0xa which is 16 plus the pre-delay ahead of R). Each op writes into an OLDER slot, so the structure is a time-reversed in-place network: a signal scattered into a slot of age A reaches an output tap of age T after T - A samples.
1. Allpass chain (g, 1 - g^2): ages 0-206 (0.75), 206-365 (0.75), 365-915 (0.625), 915-1317 (0.625), 1333-2307 (-0.7, 0.51), 8764-11374 (0.5), 16768-18109 (-0.7), 24200-28051 (0.5). The first five are input diffusion; the last three sit in the long part of the tank.
2. Modulated taps: base age 1800 (offset up to 48 samples, default rate 1.216 Hz) and base age 17500 (48 samples, 1.0 Hz): the slot at the base age is overwritten with a linearly interpolated sample from a younger slot (triangle LFO, one-sided 0 to depth).
3. Damping 3-tap filters at ages 8700 and 24150.
4. Feedback scatter ops: age 16750 receives x(1317) + t x x(16750) (a second injection of the diffused input into the middle of the tank, plus self gain t); age 1317 receives x(1317) + t x x(32754): the loop that closes the tank is 32,754 samples (0.743 s) long with feedback gain t.
5. Output taps (ages): left 18474, 22397, 26974, 30945, 5193, 9035, 12920; right 2818, 7566, 10545, 15250, 21146, 24687, 28226.
- Check against round 22d: first left output at 430 samples and right at 574 with Pre-Delay 0. The structure predicts early outputs from the late allpasses scattering into slots near a tap: left tap 18474 minus allpass end 18109 = 365 (+16 block lag and the earlier chain), right tap 24687 minus 24200 = 487, both within the measured 430 and 574 and with a fixed block offset, so the order of magnitude and the left/right asymmetry fit. A sample-level reproduction (to check the full order of ops) is not done.
- Effective decay: measured RT60 1.1 / 2.1 / 4.7 s at Time 0.25 / 0.5 / 0.75 implies a per-pass gain of 0.009 / 0.087 / 0.336 for the 0.743 s loop, close to 0.6 x t^3 to t^3; the feedback gain is t itself, so the damping and the second injection change the result. Exact decay law not derived.

## Round 27c: the seventh reverb control value (O+0x418 = fx+0x13e8) resolved
- The Reverb constructor (fcn.42013710) sets O+0x418 = 0.5 (0x3f000000), next to the other defaults (Time 0.75, Dry/Wet 0.25, Color 0.5, Pre-Delay 0.15, Mod Depth 0.2, Mod Rate 0.294 in the constructor; the parameter table defaults replace these on load). No parameter id maps to fx+0x13e8 (the 75 FX descriptors end at id 74 = Pre-Delay at +0x13dc... Mod Rate +0x13e4) and a search of the decompile finds no other store to it (the zeroing at ALL.c line 51544 belongs to a different object, not the FX struct).
- So f = 0.5 always: the control writes C+0x14 = 2f = 1.0 and C+0x10 = 2 - 2f = 1.0, which are also the constructor values. They look like left/right level gains, but the reverb core (fcn.4202b018, fcn.4202aa9c, fcn.4202af24, fcn.4202ace0) never reads C+0x10 or C+0x14. A fixed, unused leftover; no effect on the sound.

## Round 28 (LED 8, chorus Dry/Wet and filter placement)

LED 8 (answered from code, confirmed with one emulator write watch). The LED setter is fcn.4201da80(ui, id, on): writes board + (id+22)*16 + 12 (= LED entry mode at Board+0x16c+16*id) with 1 if `on` is non-zero else 0. Its direct-store search failed earlier because every LED write goes through this function with a constant id (linear-sweep grep also skips code; use the per-function r2 dump instead). Callers: fcn.42023960 (layer LED update, called per layer change) sets LED 2k = layer k active (only in Multi, from the Active flag of layer k), LED 2k+1 = layer k selected, LED 9 = (LayerMode == 4, Multi) and finally LED 8 = constant 0 (call at 0x420239ce, id 8, flag 0). fcn.4201da80 has no other caller with id 8. Result: LED 8 is explicitly forced off and never lit; unused on this firmware. The other callers set LED 13 (0x420220cb, 0x42022119; Play) and ids from fcn.4201d070 (10 to 14). Emulator check: a write watch on the LED 9 mode word fired at the retw of fcn.4201da80 after pressing Multi (button 4).

Chorus (fcn.4200feb4), exact Dry/Wet law. m = clamp(Dry/Wet arg + obj+0xc + mod, 0, 1). Targets: wet = 1 - (1-m)^2 = 2m - m^2, dry = 1 - m^2. Both are ramped linearly across the 16-sample block into obj+0x1fc (wet) and obj+0x1f8 (dry); output per sample = dry * in + wet * (sum of voice taps). Block-end level compensation g = 1 - 0.13 * stageIndex (+0.2 * fb if fb > 0, + 0.25 * |fb| if fb < 0), clamped 0..1, scale = 1 - (1-g) * wet gain; the decompile shows it applied to output indices 0, 4, 8, 12 only (probably an unrolling artefact, unverified). This reproduces the measured wet ratios 0.47 / 0.78 / 0.95 (formula 0.474 / 0.778 / 0.946).

Chorus filters are NOT in the feedback loop. High-Cut (obj+0x1c) and Low-Cut (obj+0x20) are applied to the input before it is written into the delay buffers; the feedback term is feedback * (last voice tap value, not the sum) and is added after the filters. High-Cut: one-pole low-pass, a = exp(-pi * (1-hi)^3), b = 1 - a (hi = 1 gives zero gain, matches the measured silence). Low-Cut: input low-passed by the same first stage minus a one-pole low-pass with a = exp(-pi * lo^3). The filter states (+0x1d4, +0x1ec) are shared by left and right (L then R each sample), so the filter runs at an effective 2 x 44.1 kHz on the interleaved stream: corner approx 44100 * x^3 Hz with x = 1-hi or lo (measured: hi 0.75 about 700 Hz, lo 0.25 about 694 Hz, lo 0.5 about 4.4 to 5.5 kHz) and it produces some L/R crossfeed. Feedback = f * 1.96 - 0.98.

## Round 29 (phaser structure, from code: fcn.42012b18, fcn.4202a2b0, fcn.42012ac4)

Correction: the parameter called "Feedback slot" p in rounds 22 and 22b is Spread (obj+0x20, mod +0x38). Feedback is obj+0x18 (mod +0x34) and only sets the loop gain. Parameter slots: DryWet +0xc, Depth +0x10, Speed +0x14, Feedback +0x18, Center +0x1c, Spread +0x20, PhaseDiff +0x24.

Structure (both models). Per channel a cascade of N identical second-order all-pass sections, N = table 0x3c0700a8 [Stages index] = 1, 2, 4, 6, 8 (labels 2, 4, 8, 12, 16). Section 0 coefficients are computed each block and copied to every other section (40-byte records, five coefficients plus four states; left from obj+0x44, right from obj+0x2c4), so all sections of a channel have the SAME centre frequency and bandwidth. No staggering exists in the code. Left and right differ only in the LFO phase (PhaseDiff offsets the right LFO phase).
Signal path per sample and channel: x = Feedback state x (2f - 1) + inputRamp x in; cascade; feedback state (obj+0x544 left, +0x548 right) = last section output; out = wetGain x cascadeOut + dryGain x in (dry is the raw input, not including feedback). inputRamp is 1.0 except it ramps 0 to 1 over the first block after a reset (flag obj+0x58c). Dry/Wet: wet = 2m - m^2, dry = 2(1-m) - (1-m)^2 (ramped over the block).
Centre: idx = clamp(Center + Depth' x (lfo x 0.5 + 0.5), 0, 0.98) x 256 (max 250.88), linear interpolation in the table. Depth' = Depth^2 in PPG only (Depth in Nave is linear). Speed = 0.01 + 5 x s^2 Hz. LFO is a triangle (own phase accumulators 0x554 and 0x560) or the sine table.

Section type 1 = PPG label. a = PPG table value (2 sin(pi f/fs)), k = (2 - a) x Spread x a. Coefficients (c1, c2) = (2 - a^2 - k, k - 1); difference equation y = c2 x0 + c1 x1 - x2 + c1 y1 + c2 y2. Denominator z^2 - c1 z - c2 = (z^2 - 2cos(w) z + 1) + k (z - 1); the numerator is the negated reversed denominator, so each section is an exact second-order all-pass with polarity inversion (-1 at DC and Nyquist). Spread 0 gives k = 0, poles and zeros cancel, H = -1 per section (matches the measured polarity result). Pole radius = sqrt(1 - k): bandwidth grows with Spread and with frequency (k proportional to a). Phase goes 0 to -360 degrees per section.
Section type 0 = Nave label. f = Nave table value smoothed per block (0.9 old + 0.1 new, one pole per channel), r = exp(-pi (0.008 + 0.1 Spread^2)) (0.975 at Spread 0, 0.71 at Spread 1; exp via fcn.4203d800), fcn.4202a2b0 builds y = r^2 x0 - 2 r cos(2 pi f) x1 + x2 + 2 r cos(2 pi f) y1 - r^2 y2, a standard second-order all-pass (positive polarity) with poles at radius r and angle 2 pi f.

Open conflict: resolved in Round 43 (the assumption that the half-rotation point equals the pole angle was wrong for a low pole radius; the code is consistent).

## Round 30 (tremolo Smooth and PhaseDiff, from the disassembly of fcn.42013798; xtensa-esp32s3-elf-objdump from the emulator kit toolchain is used because r2 mis-decodes some instructions)

Object offsets (obj = fx+0x13fc): Depth ramp value/target/step/counter +0x50/+0x54/+0x58/+0x5c (u16 blocks), Speed +0x60, PhaseDiff +0x64, Smooth +0x6c, Shape +0x4c (int), cached shape +0x10, LFO phases +0x14 (channel 1) and +0x18 (channel 2), phase increment per sample +0x1c, sample rate +0x0c, smoothing coefficients +0x28/+0x2c (channel 1) and +0x3c/+0x40 (channel 2), smoother states +0x30 and +0x44. Modulation pointers: Depth +0x7c, Speed +0x80.

Per sample (16 per block): u = (lfo + 1) / 2 per channel; gain = ((1 - D) + D x u)^2; out = in x gain. D = clamp(Depth arg + ramp + mod, 0, 1). So gain swings between (1-D)^2 and 1; the squared law makes Depth 0.5 a trough of 0.25 (-12 dB), not 0.5. No Dry/Wet in the tremolo.
Speed: Hz = 0.1 + 39.9 x s^2 (s = clamp(Speed + mod)), increment = Hz / sample rate (obj+0x0c), same for both channels. Tempo-synced branch (needs obj+0x74 >= 0.1) stays dormant, as in Round 22 notes.
PhaseDiff: no modulation input, no smoothing. At the start of each block channel 2 phase = wrap(PhaseDiff + channel 1 phase), in cycles: 0 = both channels in phase, 0.5 (default) = opposite phase (auto-pan), 1.0 = same as 0. It is a per-block resync, so moving it makes channel 2 jump at the next block boundary.
Shapes: 0 Sine = 1024-entry table (0x3fcd65d8 entry 0) with linear interpolation; 1 Triangle = 4p for p < 0.25, 1 - 4(p - 0.25) up to 0.75, then -1 + 4(p - 0.75) (starts at 0 rising); 2 Square = +1 for phase < 0.5 else -1. (Shape branch bodies are partly garbled in the disassembly; the triangle and square forms are inferred from the surviving constants 0.75, 0.5, 4.0, -1.0 and match the chorus/flanger/phaser triangle.)
Smooth: acts ONLY when Shape is Square. It is a one-pole low-pass on the square LFO of each channel: w = 0.06 + 0.3 x (1 - Smooth), t = exp(-pi x w^3), y = (1 - t) x + t y_prev, once per sample at 44.1 kHz. Time constant 1 / (pi w^3) samples: Smooth 1.0 gives w = 0.06, about 1470 samples (33 ms); Smooth 0.5 gives w = 0.21, about 34 samples (0.77 ms); Smooth 0 gives w = 0.36, about 6.8 samples (0.15 ms). Higher Smooth = softer edges. For Sine and Triangle the Smooth knob does nothing.

## Round 31 (voice steal priority, fcn.4037a4b0, read from objdump; the decompile in ALL.c collapsed the 8-iteration search loop and is wrong for this part - round 21b and the "Voice allocation" paragraph describing a single candidate slot are superseded)

Polyphonic note-on (mono/legato flag part+0x397 = 0 and velocity > 0). The note state table (part+0x62c+note) is set to 1 first. Then the code walks the 8 voice slots (stride 0x70, state byte at slot+0x11) in round-robin order starting at (lastStarted + 1) mod 8, with a ninth step that decides:
1. First FREE slot (state 0) in that order is used immediately (voice trigger fcn.42014b18, lastStarted = that slot), but only if the number of non-free slots (fcn.4204e808 counts state != 0 over all 8) is below the part polyphony limit (part+0x392). When the part is AT the limit, free slots are skipped and the walk continues, so the limit is enforced by stealing.
2. States seen while walking: 1 = note held, 2 = releasing (set by the all-notes-off code with voice vtable +0x14), 3 = fast fade because of a steal or all-sound-off (voice vtable +0x1c). Per slot the walk records candidates: (a) state 2: the one with the smallest 64-bit stamp at slot+0x28/0x2c (release stamp); (b) state 1: smallest stamp at +0x20/0x24 (trigger stamp); (c) state 3: smallest +0x28/0x2c; (d) states 1 and 3: the voice with the HIGHEST note (16-bit value at slot+0x12, strict greater-than, so the first in round-robin order wins ties).
3. After the walk, if no free slot was used: first a RELEASING voice (a) is stolen if one exists. Otherwise, with the byte part+0x391 equal to 0, the voice with the highest note among held and fading voices (d) is stolen. If part+0x391 is non-zero (no writer found, so presumably always 0; the byte is zero-initialised) the oldest held voice (b) would be stolen, else the oldest fading voice (c).
4. Steal action: state := 3, voice vtable +0x1c (fast fade), stamps +0x20/0x24 and +0x28/0x2c := part stamp (part+0x3a0/0x3a4), pending note (note | previous note << 16) at +0x30, pending velocity at +0x34, lastStarted := that slot. The new note starts when the fade finishes (start code not traced). Stealing a voice that is already fading (state 3) replaces its pending note, so the earlier pending note is lost.

Consequences (code only, not verified in the emulator or on hardware): with all voices held the new note takes over the HIGHEST sounding note, not the oldest; releasing voices are always sacrificed first. The stamp pair part+0x3a0/0x3a4 is written only with 0 (constructor, reset at 0x420149fc, 0x4201639c) in the code I found; if nothing else advances it, all stamps are equal and ties resolve to the first slot in round-robin order, which makes rule (a) "first releasing voice after the last started one" rather than "longest released". Open items closed in Round 44: the stamp IS advanced (every block), so the 'all stamps equal' remark above is wrong; fade length and pending-note start are described there.

## Round 32 (low-pass filter structure, types 0 = LP24 and 1 = LP12; from the render kernels at 0x4037e291 and 0x4037e30c, the coefficient code at ALL.c ~3480-3515, and live constants read from a sounding voice)

Structure (both low-pass types). A four-stage zero-delay-feedback (trapezoidal, TPT) ladder: four one-pole low-pass stages in series, y_i = s_i + g_i (y_{i-1} - s_i), s_i = y_i + g_i (y_{i-1} - s_i). Stages 1 and 3 use g_a (voice+0x410), stages 2 and 4 use g_b (+0x414). Feedback comes from the four states, not from the output: u = (x - (w1 s1 + w2 s2 + w3 s3 + w4 s4)) x K, with weights w4 = k (1-g_b), w3 = k g_b (1-g_a), w2 = k g_a g_b (1-g_b), w1 = k g_a g_b^2 (1-g_a) (voice+0x434, 0x430, 0x42c, 0x428) and K = 1 / (1 + k (g_a g_b)^2) (+0x43c). That is the exact algebraic solution of the delay-free loop for feedback gain k = 2 x Resonance (Resonance x 0.8 for type 1). The loop input is soft-saturated before stage 1: s(u) = u / (|u| + 0.4348) (constant +0x3e8 = 1/2.3, reciprocal by a rsqrt magic-number guess plus one Newton step), so the small-signal gain of the input is 2.3 and the maximum is 1.
Type 0 (Low-Pass 24dB): output = y4 x M. Type 1 (Low-Pass 12dB): same ladder, same feedback weights, output = y2 x M (tap after the second stage). M (+0x438) = 2 x Resonance x 2.0 + 1 = 1 + 4 x Resonance (constant +0x3ec = 2.0), so Resonance 1.0 gives M = 5. The ladder's own DC gain is 1/(1+k), so the net DC gain at full resonance is (1+4R)/(1+2R) = 1.67 for type 0 (bass is boosted, not lost). Output gain after the filter 0.75 (older notes).
Cutoff law. G0 = 0.000582427 x 2^(1.66667 + 10.4167 x c), c = cutoff after modulation, limited to 0.9 when Resonance > 0.62. Stage values G_a = 0.865 x G0 and G_b = 1.135 x G0 (constants +0x3f4, +0x3f8: the stages are deliberately detuned by about -13.5 percent and +13.5 percent). g = G / (1 + G). G is used directly as the TPT warped frequency, so the stage corner is fc = (44100 / pi) x atan(G): c = 0 gives about 23 Hz (stage a) and 30 Hz (b); c = 0.3 about 195 / 255 Hz; c = 0.5 about 830 / 1090 Hz; c = 1.0 about 16.0 / 17.3 kHz. The earlier "g = tan?" question is answered: 0.000582 x 2^x is G (the tan value), not g. Live check on a sounding voice at cutoff 1.0: g_a 0.6856, g_b 0.7383 (G 2.18 and 2.82, ratio 1.29 versus 1.31 expected from the stage constants).
Live constants read from a voice (layer A, voice 1): +0x3e4 2.3, +0x3e8 0.434783, +0x3ec 2.0, +0x3f0 0.27 (use not found), +0x3f4 0.865, +0x3f8 1.135, +0x3fc 0.000188395 (use not found), +0x400 128 (use not found), +0x404 10.4167, +0x408 1.66667, +0x40c 0.000582427. Script scripts/emu_tests/filt_rd.py (reads firmware data only, no audio).
Round 21c puzzle (LP24 "flat spectrum at cutoff 0.3"): the model predicts stage corners near 200 Hz at c = 0.3 with 24 dB per octave, so a note around 130 Hz would lose almost everything above its first harmonics; a flat reading in the earlier test is therefore most likely the noise floor of the recording, not a filter fault. Not re-measured.

## Round 33 (Flavour consumer search, static, exhaustive over the per-function dump)

Result: Flavour (engine id 0x14, array slot engine+0x333c) has no consumer in v1.02. This upgrades the Round 15 conclusion from "none found" to a closed enumeration.
- Every access to the engine region 0x3200..0x33ff uses the base forms addmi +0x3200 / +0x3300 (12 sites, all inside the engine at 0x420146c2..0x42015783). Offsets used after them: 36 (LayerMode, 0x3324), 40 (LayerAlloc, 0x3328), 232 and 224 (state bytes 0x32e8 / 0x32e0), 236 (0x32ec, array base in setParam fcn.420157e0). Offset 60 (0x333c) is never used; no movi of 0x333c, 0x32ec or 0x3300 exists either.
- The generic engine getter fcn.42014cd0 has no direct caller, and its note-processor wrapper fcn.4204e5a4 (vtable 0x3c070110 + 0x3c, passes the id to engine vtable +0x3c) is referenced only from that vtable. Virtual calls through vtable offset 60 exist at five places (0x40387f13, 0x403885ab, 0x4038bf24, 0x4038cc84 in an IRAM I/O class with 3 to 4 arguments, and 0x4203768f which is a field read, not a call); none passes id 0x14 or matches the getter's single-argument call.
- The note processor setter (fcn.42013f40) has no case for 0x14, and the only code that mentions the UI id 0x22f is the Master page handler (fcn.42021420 area: reads the pot, stores the Sound value, pushes it with fcn.42019024, displays it).
Consequence for the improvement list: the Flavour pot, preset slot, display line and the engine array slot already exist and are fully wired up to engine+0x333c, so a patch only needs to add a consumer (one float load at engine+0x333c, i.e. addmi 0x3300 then lsi at offset 60, in the code that should use it).

## Round 34 (delay object +0x34 and the stereo routing, from fcn.42010ba0 and its constructor fcn.420113e0; impulse check done, see the end of this round)

Object +0x34 (fx+0x718) is a hidden cross-feed ("ping-pong") amount. The constructor sets it to 0.7 (0x3f333333); no parameter, setter or modulation input writes it (the Delay parameters 59 to 68 skip fx+0x718), so it is a constant 0.7 in v1.02. Used as X = 1 - (1 - p)^4 (two squarings of 1 - p) = 0.9919 at p = 0.7.
Per-sample routing (L = left, R = right; echoL/echoR = the delayed, filtered signals read from the left and right buffers; fb = Feedback^2, Feedback clamped 0..1 first):
- left buffer write = inL x (1 - X/2) + inR x X/2 + fb x (X x echoR + (1 - X) x echoL)
- right buffer write = inR x (1 - X) + fb x (X x echoL + (1 - X) x echoR)
- output L = dry x inL + wet x echoL, output R = dry x inR + wet x echoR (outputs are not crossed).
With X = 0.992 the input is mixed to the left buffer only (0.504 L + 0.496 R), the right buffer gets 0.8 percent of inR directly, and each echo bounces to the other channel: first echo on the left at Left time, first echo on the right at Left time + Right time, then left again after Right + Left, and so on. So Right is the time of the bounce into the right channel, not an independent right tap. Feedback is applied to the cross-fed signal. Buffers are clamped to +-100.
Echo filtering per channel (same coefficients both channels, separate states): a first-order recursive section y = b1 x + b2 x1 + b0 y1 with (b0, b1, b2) = (0.99, 1.0, -1.0) at +0xb0 / +0xb4 / +0xb8 (constants 0x3f7eb852, 0x3f800000, 0xbf800000: a DC blocker with corner about 70 Hz), then High-Cut one-pole low-pass (t = exp(-pi (1 - HighCut)^3), as in the chorus) and Low-Cut as low-pass subtraction (t = exp(-pi LowCut^3)). These sit inside the feedback loop (they act on the signal read from the buffer, which is also written back), unlike the chorus.
Time: Link (obj+0x44 > 0.1) makes Right time = Left time + obj+0x40 x 4410 samples (obj+0x40 is 0 and never written, so equal). Time is clamped to 1..88,198 samples; changes slew by 0.000125 of the difference per sample (see the Delay section above).

Impulse check (unmodified firmware, impulse 0.25 on both inputs, FX1 = Delay, free time, Link off, Left 0.3, Right 0.4, Feedback 0.7, Dry/Wet 1.0, High/Low-Cut 0; scripts/emu_tests/dl_run.py). Expected: Left time 7,937 samples, Right time 14,113 samples. Measured peaks (sample, value): left output 8,083 (0.198), 30,439 (0.026), 52,730 (0.005); right output 22,404 (0.065), 44,711 (0.011). So: first left echo at about the Left time, first right echo at about Left + Right (22,050 expected), second left echo one Left time after that (+8,035), then right again (+14,272). There is no right echo at the Right time alone (the 0.8 percent direct path would be 0.002, below the 0.004 detection threshold). The extra 150 to 350 samples per hop match the slewed read pointer plus the 16-sample block pipeline; the unexplained 15 ms in Round 9 is consistent with this. Amplitudes: right first echo / left first echo = 0.33 against Feedback^2 x X = 0.49 x 0.99 = 0.49 expected from the formula alone; the missing factor 0.68 is not explained (filter smearing of the one-sample impulse across several samples is the likely cause; peak amplitude of a smeared impulse is not a gain measurement). The routing (ping-pong with Right as the bounce time) is confirmed; the exact gains are not.

## Round 35 (phaser frequency tables, built by the phaser constructor fcn.420132bc, loop at ALL.c ~46500)

Both tables have 256 entries, filled once at construction with i = 0..255, x = i / 256:
- Nave table (0x3fca5014): f/fs = 11.56232572 x 2^(10.66667 x x) / 44100 (base constant 1625.4987 = 2^(32/3)). 11.5623 Hz = 440 x 2^(-63/12), so entry i is the pitch of MIDI note 6 + i/2: half-semitone steps, 128 semitones over the table. Entry 0 = 11.56 Hz, entry 128 = 466.16 Hz (note 70), entry 255 = 18,259 Hz.
- PPG table (0x3fca541c): 2 x sin(pi x Nave[i]), the Chamberlin state-variable coefficient of the same frequency (1.9275 at entry 255).
Lookup (fcn.42012ac4): index = clamp(Center + Depth' x (lfo x 0.5 + 0.5), 0, 0.98) x 256, linear interpolation between neighbours, so Center is a pitch scale: Center c gives MIDI note 6 + 128 c at LFO centre. Center 0 = 11.6 Hz, 0.25 = 73 Hz, 0.5 = 466 Hz, 0.75 = 2.96 kHz, 0.98 (the clamp) = about 16.2 kHz. The left and right channels use two separate lookups from the two LFO phases.
Constructor defaults of the phaser (object offsets): DryWet 0.5, Depth 0.3, Speed 0.5, Feedback 0.6, Center 0.5, Spread 0.7, PhaseDiff 0.5, Stages index 2 (8 stages), LFO shape 0 (Triangle), Model -1 (forced to be set at first block).

## Round 36 (preset migration fcn.420294f0; read from objdump, the decompile dropped the mode argument)

Signature: fcn.420294f0(this, index, value float, mode). The loader (fcn.420298f4, call at 0x420299c8; also 0x42029b80) passes mode = byte 0x19 of the 32-byte sound header, which is the sound FORMAT VERSION. A new or empty slot is written with 8 (fcn.42029a78 area, ALL.c 74772 and 74860), so the current version is 8 and mode 8 or higher changes nothing. After loading, the version byte in the sound object (+0x8dd) is set to 8, so a re-saved preset is current. Indices above 0x14f are returned unchanged. s = index mod 0x54 (UI index within the layer, ui_params.txt order); n = trunc(value + 0.05).

Operations (names by UI index):
- A. Mod-matrix Source (s = 0x3a + 3k, k = 0..7): if n = 5 add 2, if n > 5 add 4 (inserts LFO1*MW, LFO1*AT and LFO2*MW, LFO2*AT; old 5 = LFO2, old 6 = Mod Wheel). Then B-src below.
- B-src. Source, thresholds on the already shifted value: n > 11 add 1, then n > 14 add 1 (inserts Note PB at 12 and MPE Y at 15).
- C. Mod-matrix Target (s = 0x3b + 3k): n > 9 add 1 (inserts Resonance at 10).
- D. Wtb1Table (s = 4) and Wtb2Table (s = 11): n > 55 subtract 24 (old presets stored the table number, new ones store the step; steps 32..63 map to tables 56..87).
- E. Envelope times Attack, Decay, Release of the Wave env (s = 19, 20, 22), Amp env (32, 33, 35) and Filter env (37, 38, 40); Sustain is not touched. idx = clamp(trunc(value x 127), 0, 127), t = T[idx] with T the 128 integer table at 0x3c075f30 (identical to the ROM table 0x3c070340 that builds the live coefficient table c[] at 0x3fccde98). If t is 0 or above 32766 the value is kept. Else x = t / 32767 and the result is (j - 1) / 127 for the first j = 1..127 with c[j] <= x (c[j] = T[j] / 16 / 32767, c[1] = 1.0); no match gives 1.0. So the per-tick coefficient is kept and the index moves down: old 42 becomes 2, 60 becomes 19, 80 becomes 40, 127 becomes 87; old indices up to 41 all become 1 (the fastest non-instant step). Old indices 0 and 1 stay instant.
- F. LFO1 Speed (s = 0x2b) and LFO2 Speed (s = 0x30): value = sqrtf(value) through fcn.4203d928 (a wrapper that returns NaN and sets errno 33 for negative input). The live law is Speed squared, so the old linear speed is kept.
- G. FilterType (s = 0x1d), enum remap. Version 6 and below (old order LP, HP, Bypass, Drive): 0 to 0, 1 to 2, 2 to 5, 3 to 4. Version 7 (order LP, HP, Drive, Bypass): 0 to 0, 1 to 2, 2 to 4, 3 to 5. The current order is 0 LP24, 1 LP12, 2 HP24, 3 HP12, 4 Drive, 5 Bypass, so the 12 dB types are new in version 8.

Which version does what (mode = stored version):
| version | applied |
|---|---|
| 0, 1 | A, B-src (Source slots), then everything for version 2 |
| 2 | D, E, F, C, B-src, G |
| 3 | F, C, B-src, G |
| 4 | C, B-src, G |
| 5 | B-src, G |
| 6 | G (old order) |
| 7 | G (Drive and Bypass swapped) |
| 8 and up | none |
Sources: dispatch at 0x4202953b (mode < 2), 0x42029574 (2), 0x4202965c (3), 0x42029675 (4), 0x420296dc (5), 0x4202975d (6, 7).
Consequence for improvements: a rebuilt firmware that adds enum entries or reorders a stored parameter must bump the version byte (8 to 9) and add a case here, or old presets will load with shifted values. Not yet confirmed on hardware.

## Round 37 (LFO skew writers found; this closes the open item from rounds 9, 26 and 27)

Both skew fields are initialised to 0.5 by constructors and never changed afterwards:
- Voice LFOs: the voice constructor fcn.420162d0 (ALL.c 50052) loops over all 32 voice objects (4 layers x 8, base 0x3fcaaf78, stride 0x828, end 0x3fcbb478). In it puVar2 = voice + 0x400, and `puVar2[0x5c] = 0x3f000000` is voice+0x570 (LFO1 skew, LFO object at +0x560) and `puVar2[99] = 0x3f000000` is voice+0x58c (LFO2 skew, LFO object at +0x57c). The same loop sets 0.5 at voice+0x4e0 and +0x4e4 (puVar2[0x38], [0x39]; the ramped value of per-part k=0x35). Emulator check (scripts/emu_tests/skew_rd.py): all 8 voices of layer A read 0x3f000000 at +0x570 and +0x58c.
- Global LFOs: the engine constructor fcn.42016a30 (loop of 4 parts, step 0x4074 from 0x3fcc1ce8) stores 0x3f000000 at +28 and +56, which are 0x3fcbdcc4 + part x 0x4074 + 0x4040 and + 0x405c, the skew of global LFO1 and LFO2.
So the phase warp is always symmetric (skew 0.5): the LFO shapes are the plain waveforms. No parameter, mod-matrix route or pot changes it in v1.02. A rebuild could expose it by writing the float at voice+0x570 and +0x58c, or at the global fields, but no UI parameter exists for it.

## Round 38 (the three "display helpers" on the open list; one of them is not a display function)

- fcn.420284dc (889 bytes) is the KNOB ADC SCAN TASK, not display code. It creates an ESP-IDF continuous ADC (12 bit, sample rate 0x4b00 = 19,200 Hz, frame size 0x100 bytes, 4 channels listed as 1 to 4) with a calibration scheme, and two GPIO outputs (GPIO 15 and 16, written directly through the GPIO set/clear registers 0x60004008/0x6000400c with bits 0x8000 and 0x10000) as a 2-bit multiplexer address. Each pass switches the mux (bit 0 of the step to GPIO15, bit 1 to GPIO16), reads a DMA frame, and files each sample by its channel number: pot index = mux step x 4 + (channel - 1), so 4 mux positions x 4 ADC channels = 16 pots. 16 samples per pot are summed and divided by 16 (a 12 bit average), then passed through a smoother and stored as u16 into the pot array (arg1, 16 entries; a second array of raw averages is kept on the stack). The array is what the knob scan (fcn.4201ea04, round on Pot Settings) reads.
- Knob smoother (fcn.42028960 constructor, fcn.420289b8 step; one object of 16 bytes per pot, built with 2.0, 75.0, 0.01): two cascaded one-pole filters s1, s2 with a speed-adaptive coefficient a = min(1, a0 + 0.04 x |s1 - s2|), s1 += (x - s1) a, s2 += (s1 - s2) a, output s2. a0 = 2f / (1 + f) with f = tan(pi x 2 / 75) (value about 0.155). So slow, small movements are smoothed hard and a change of about 25 counts or more passes almost directly (a reaches 1).
- fcn.42028c40 is the INVERSE text draw into the 1-bit frame buffer: glyph pixels are cleared and the rest of the character cell and the spacing columns are set; its sibling fcn.42028e0c draws normal text (glyph bit set sets the pixel). fcn.42028f30(.., text, invert) measures the string (fcn.42028ef8, sum of glyph width + spacing from the font vtable) and calls one of the two. The frame buffer is 128 rows of 8 bytes (x clamped to 63, rows to 127), so the 128x64 OLED is stored transposed. Font object vtable: +8 height, +0xc glyph width, +0x10 column bits of a glyph, +0x14 spacing.
- fcn.4201eea8 is the horizontal VALUE BAR widget: label string as the last argument (call sites with "Volume" and "Flavour" at ALL.c 62970 and 62976, and the generic one in fcn.4201f00c), value 0..1, a bipolar flag. Unipolar: bar from the left edge to value x width. Bipolar: bar between the centre and the value (value 0.5 = empty). Two layouts chosen by a flag: full width 0x80 pixels, or 0x6c pixels at a 10 pixel offset. The fill is drawn by fcn.4204f804 and the label by fcn.42028f30.

## Round 39 (UI open items closed: page identities, Info, debug page 11)

- Page objects: all identified in Round 25b and 25c (states 3 settings list, 4 Play/arp note page, 5 sequencer record/edit, 6 sequencer step values, 7 sound list, 8 Multi/Single, 9 Info, 10 Mappings, 11 debug). The "still unnamed" wording in older lines is stale.
- Info (state 9): shows the title and the version constants 1, 3, 3 ("Version 1.03 b3"), see Round 25. Nothing else dynamic found.
- Debug page (state 11, vtable 0x3c073830, draw fcn.420244c8) shows two timing lines in the format "0: %.1f < %.1f" and "1: %.1f < %.1f", each as AVERAGE < PEAK of a processing time (almost certainly microseconds from the esp timer, unit not verified):
  - Line 0: the engine audio routine. Object 0x3fca3974 +0 = running average (0.95 x old + 0.05 x elapsed), +4 = peak; the peak is read and cleared by the page each redraw. Written at the end of the routine at 0x4200d4b4..0x4200d4f6, gated by the flag byte at object+0x1698.
  - Line 1: the engine task loop fcn.4200ccc8 (wakeup, handling of queued events, then done). Average at 0x3fca37a4 and peak at 0x3fca37a0 (read and cleared by fcn.4200d51c), same 95/5 filter, gated by the byte at 0x3fca37a8. During the handling the task drives GPIO 21 high (fcn.42026b80(1)) and low afterwards, a scope marker for the busy time.
  - The gate flags are set by controller event 0x23 through fcn.4200d4fc (one call sets both bytes). Entering the debug page posts event 0x23 with value 1 (fcn.42024490), leaving posts 0 (fcn.420244ac); a serial console command (character 0x4c) also posts it with 0.
  - Consequence: the flags default to 0 (cleared in the constructor), and state 11 cannot be reached in v1.02 (Round 25c), so the load meter is never on in a normal unit. A rebuild could expose the page, for example behind a button combination, to read the real CPU load of the audio routine against its 16-sample block budget of about 363 us (16 / 44100). Not verified on hardware.

## Round 40 (reverb feedback routing; read from objdump only, no emulator data used as a source)

Corrections to Round 27 and 27b (the decompile hid the 16-sample loops; every op below is a loop over 16 consecutive slots per block, R = read head C+8, ages are R minus index, 65,536 slot ring):
- Order per block (fcn.4202b018): write the mono input block (inL + inR) at W, run the 3-tap pre-filter, then fcn.4202aa9c once: (1) 8 allpass records (A, B, g1, g2): Y = buf[R-B+j], X = buf[R-A+j]; Y' = g1 X + g2 Y; X' = X - g1 Y. (2) 2 modulated taps. (3) 2 damping filters. (4) 2 recirculation records. Then fcn.4202af24 sums the 7 + 7 output taps (out[j] += gain x buf[R - age + j]). R and W advance by 16.
- Damping filter (ages 8700 and 24150): in place and recursive over the slots: y[n] = t c1 x[n] + k y[n-1] + t c2 x[n+1], with (k, c1, c2) from Color. At Color 0.5: k = -0.99, c1 = c2 = 0.995, so the DC gain is exactly t. (Round 27 called these "outer taps scaled by t", which is right, but the filter is a one-pole IIR, not a 3-tap FIR.) The pre-filter uses the same coefficients without the t factor (DC gain 1).
- Recirculation records are (L1, L0, L2): buf[R-L1+j] = buf[R-L0+j] + t x buf[R-L2+j].
  - Record 0 = (16750, 1317, 16750): copies the diffused input from age 1317 into the slot at age 16750 and multiplies the signal passing age 16750 by t.
  - Record 1 = (1317, 1317, 32754): adds t x the slot at age 32754 into the slot at age 1317.
  - A slot moves one age per sample, so a copy from age s to age d changes the delay by s - d. Record 1 therefore closes a loop of 32754 - 1317 = 31,437 samples (0.713 s), NOT 32,754 as written in Round 27b. A trip round the loop passes record 0 (x t), damping at 8700 and 24150 (x t each at DC) and record 1 (x t): about t^4 per trip at Color 0.5. Allpasses are unity gain.
- Modulated taps: only record 0 (age 1800) is driven by the Mod Depth and Mod Rate parameters (depth target = 48 x ModDepth x 5 samples, rate = 1.2158 x ModRate^3 x 20 Hz, smoothed 1 percent per block). Record 1 (age 17500, inside the tank) has fixed constants from the constructor: depth 48 samples, rate 1.0 Hz. So "Mod Depth 0" is NOT static (Round 22d assumed it was): the tank tap at 17500 keeps wobbling. Read from code only, not measured.
- Write position and Pre-Delay (fcn.4202aef4, called from the control when a parameter changed): P = 16 x (min(trunc(pd x 16370 + pd^2 x 16370), 0x7fe0) >> 4), so P is a multiple of 16 samples from 0 to 32,736 (0.742 s). It sets W = R + 16 + P directly (no crossfade; the ring is not cleared by it). The constructor starts with R = 0, W = 16 (pd 0). The pre-filter offset (core +0x3c0) is written only by the constructor, as 0, and nothing else in the reverb writes it, so the 3-tap pre-filter always runs on the 16 slots W-1 .. W+14, that is the block just written (plus one slot of the previous block). Because every network op works at ages relative to R, which trails W by 16 + P, an input sample is first touched by the network one block (16 samples) after it was written, plus P.
- Modulated taps (both records): the LFO phase starts at 0 and the rate and depth values start at the constructor values (record 0: 48 samples depth and 1.2158 Hz until the control smooths them to their targets at 1 percent per block; record 1: 48 samples and 1.0 Hz for good). Per block each tap overwrites ONE slot, the one at age base (1800 or 17500), with a linear interpolation between two younger slots (offset 1 + depth x triangle samples); there is no 16 slot loop for this op.
- A Python model built only from these ops (scripts/models/reverb_model.py; Color 0.5, Pre-Delay 0) gives, from the code alone: first left output 398 samples and first right output 542 samples after an impulse at the start of a block (not counting any I/O buffering outside the reverb), per-trip loop gain close to Time^4, and no decay at Time 1.0. It is an aid for reading the code, not evidence from the hardware. Whether it matches a real unit is not verified.
- Open from code: the exact tap gains for Time near 0.5 depend on the smoothing of the Time value (1 percent per block), which the model holds constant; the second modulated tap's effect on the tail is unquantified.

## Round 41: flanger, read from objdump only (no emulator data used as a source)
Functions: process fcn.42012318 (vtable +0x10), reset fcn.420122d8 (vtable +0xc), constructor fcn.42012978, buffer setup fcn.42012a20. Object at part+0x98c, size 0xb4. Block = 16 samples, LFO rate constant 2756.25 = 44100/16 (field +0x84).
- Parameter IDs and fields: 0xd DryWet +0xc, 0xe Depth +0x10, 0xf Speed +0x14, 0x10 Feedback +0x18, 0x11 Center +0x1c, 0x12 PhaseDiff +0x20, 0x13 Shape +0xa4 (int = trunc(v + 0.05); 0 = triangle, any other value = table mode).
- DryWet: m clamped 0..1; wet = 2m - m^2; dry = 1 - m^2 (exactly 2(1-m) - (1-m)^2). m < 0 gives wet 0, dry 1. Both gains ramp linearly across the 16 samples from the previous block value (first sample already takes 1/16 of the step). Reset sets dry 1, wet 0.
- Depth D = clamp(d, 0..1)^4 x 1411.2 samples (0 if d < 0). Speed S = clamp(s, 0..1)^4 x 20 + 0.001 Hz (0.001 Hz if s < 0). Speed 0 is therefore 0.001 Hz (period 1000 s), not stopped. Phase step per block = S / 2756.25. Center C = 1 + c^2 x 1102.5 samples.
- Delay per channel = C + (0.5 + 0.5 x lfo) x D, clamped to 0..2612. This is additive after the square law (Round 22f guessed a squared sum; superseded). Range is C to C + D, at most 1103.5 + 1411.2 = 2514.7 samples, so the 2612 clamp is never reached.
- LFO: triangle (phase p: 4p up to 0.25, 1 - 4(p - 0.25) to 0.75, then -1 + 4(p - 0.75)) or table mode (1024-entry sine table, entry 0 of the array at 0x3fcd65d8, index truncated, no interpolation, phase in table units). Right phase = left phase + PhaseDiff, wrapped. The LFO phase is not touched by reset.
- The delay is computed once per block and is constant over its 16 samples (no per-sample ramp). At Speed 1 and Depth 1 the delay can step by up to about 20 samples per block (arithmetic from the formulas: 20 Hz / 2756.25 x slope 4 x 0.5 x 1411.2). Not checked against hardware.
- Delay line: separate L and R rings of 2613 floats (0xa35, 0x28d4 bytes each), allocated in the part setup with fcn.403798c8(0x28d4, 0x400) twice (unconfirmed that this is heap_caps_malloc with the PSRAM flag) and zeroed by fcn.42012a20. Read before write, linear interpolation: out = (1 - f) x buf[w - d] + f x buf[w - d - 1] with d = trunc(delay), f = fraction. Since Center >= 1 the minimum delay is 1 sample.
- Signal per channel: buf[w] = in + fb x delayed; out = wet x delayed + dry x in. So H(z) = dry + wet z^-D / (1 - fb z^-D). No filter and no clip inside the loop in this function.
- Feedback: x clamped 0..1 (x < 0 gives fb = -1), v = 2x - 1; v >= 0: fb = 1 - (1 - v)^3; v < 0: fb = (1 - |v|)^3 - 1. Range -1.0 to +1.0, zero at x = 0.5, reaching exactly +-1.0 at the ends. The 0.976 limit in Round 22e came from the emulator and is not in the code.
- Block gain after the loop: g = 1 - wetEnd x 0.45 x |fb|, applied to all 16 samples of both outputs (skipped when exactly 1). Example: Mix 1 and |fb| 1 gives 0.55.
- Reset (fcn.420122d8, called from the part reset fcn.4200f82c): dry 1, wet 0, flag +0xb0 = 1, both flush flags set, flush base = current write position, counters 0. Flush clears 32 slots per sample (2614 slots, 82 samples = 1.9 ms) and forces the delayed value to 0 meanwhile. Same for L and R. Flag +0xb0 is cleared at the end of every block; no reader found in the process function.
- Modulation pointers +0x24 (mix), +0x28 (depth), +0x2c (speed), +0x30 (feedback) are added when non-null. The constructor sets them to 0 and grep of the decompile finds no writer (not confirmed in the disassembly). Both call sites in the FX chain function pass 0.0 for the two extra offset arguments (mix and depth).
- Full-Depth left dropout (Round 22f, emulator): the code has no path that zeroes one channel except the 82-sample flush after a reset. With Center 0.5 (277.4 samples) and Depth 1 the left delay reaches 1688.6 samples, beyond a 100-block (1600-sample) window, and the right delay is then about 366 samples. So a missing left response in such a window is consistent with the delay, not with a dropout. Unconfirmed, code reading only.
- Open: sine table amplitude range assumed -1..1 (table contents not read here); +0xb0 reader; who writes the four modulation pointers (if anyone).

## Round 42: delay right/left echo level ratio, read from the decompile and disassembly of fcn.42010ba0 only (no emulator data used as a source)
- Both echoes are the buffer reads after the same filter chain (DC blocker 0.99, High-Cut low-pass t = exp(-pi (1 - HighCut)^3), Low-Cut high-pass t = exp(-pi LowCut^3)), each channel with its own state. The filtered echo is what goes to the output (wet x echo + dry x in) and what is fed back, so output and feedback carry the same level.
- Per hop gain = fb x X, with fb = Feedback^2 (Feedback clamped 0..1) and X = 1 - (1 - 0.7)^4 = 0.99190. Echo order: L1 (Left time), R1 = fb X x L1 (Left + Right time), L2 = fb X x R1, and so on. So for passband content the first right echo is fb x X = Feedback^2 x 0.9919 times the first left echo, and every later echo is again fb x X times the previous one. The left channel carries the odd echoes and the right channel the even ones.
- Examples: Feedback 0.75 (the embedded default sound) gives 0.558; Feedback 0.7 gives 0.486.
- Filter loss per pass on a single-sample impulse with High-Cut 0 and Low-Cut 0: peak gain 0.957 (High-Cut 0 gives t = 0.043; the DC blocker leaves the peak essentially at 1). Applied to a one-sample impulse the peak ratio becomes 0.465 (Feedback 0.7) and 0.534 (Feedback 0.75). This is computed from the coefficients, not measured. The Round 34 impulse measurement is emulator data and is set aside; peak height of an impulse also depends on the fractional read position, so it is not a gain figure.
- DC content does not pass (the blocker has zero DC gain), so the ratio applies to anything above the blocker corner (about 70 Hz).
- Left buffer input is 0.504 x inL + 0.496 x inR (equal levels for a mono input); the right buffer gets only 0.0081 x inR directly, so there is no separate right tap at the Right time.
- Open: effect of the fractional read position and the 0.000125 slew on short transients (code reading only); the cut-off function fcn.4203d800 was assumed to be exp as in the chorus.

## Round 43: phaser Nave Spread, resolved from the code (coefficient builder fcn.4202a2b0 and the Nave branch near ALL.c 46006 read; no emulator data used as a source)
- Code facts: Nave section = all-pass with numerator r^2 - 2 r cos(th) z^-1 + z^-2 and denominator 1 - 2 r cos(th) z^-1 + r^2 z^-2, th = 2 pi f (f from the Nave table, smoothed 0.9 old + 0.1 new per block), r = exp(-pi (0.008 + 0.1 Spread^2)). Stored coefficients: +4 = 2 r cos(th), +8 = -r^2, +0xc = r^2, +0x10 = -2 r cos(th), +0x14 = 1. All sections of a channel share these values, so Spread does not stagger frequencies; it only changes r (0.975 at Spread 0, 0.948 at 0.3, 0.836 at 0.7, 0.712 at 1.0).
- Why the half-rotation frequency still rises with Spread: for a second-order all-pass the phase is -pi at the pole angle only when r is close to 1. With a smaller r the -pi point moves above th, because the phase curve is not symmetric on a linear frequency axis. Computed numerically from the transfer function above (total rotation -2 pi from DC to Nyquist, checked): with th at 466.2 Hz (Center 0.5) the -pi point is 498 Hz at Spread 0, 598 Hz at 0.3, 1334 Hz at 0.7, 2382 Hz at 1.0. With th at 1350 Hz it is 1361, 1400, 1835, 2687 Hz. A cascade of identical sections has its half-rotation at the same frequency.
- So the code gives a rising half-rotation point with Spread. The Round 29 statement "exactly at the pole angle, independent of Spread" was wrong, and "stagger" in Round 30 is the wrong word: it is a bandwidth effect, the same as in PPG. The conflict is closed without any extra test.
- Open: none for this item. The numbers are computed from the transfer function, not measured on hardware.

## Round 44: voice steal stamp, fade and pending-note start (objdump of 0x4037a794, 0x4037a92c, 0x4037bfb0 gate, 0x42016280, 0x4204e87c; no emulator data used as a source)
- Stamp pair: the note-processor render function (fcn.4037a92c, the one that calls the 8 voice slots) ends with a 64-bit add of 16 to part+0x3a0/0x3a4 (0x4037ac29 to 0x4037ac3c, carry via saltu). So the stamp is a sample counter that advances by 16 per block. It is passed to every slot call (fcn.4037a794) as "now". Round 31 said it is only ever written with 0: that was wrong. Trigger stamp (slot+0x20/0x24 in Round 31 numbering) is stored by the trigger fcn.42014b18; release stamp (slot+0x28/0x2c) when a slot goes to state 2 (seen in fcn.4037a794 at 0x4037a915). Oldest-first and longest-released-first therefore work as written in Round 31; ties only occur for events in the same block.
- Slot per-block function fcn.4037a794 (offsets relative to the slot object, i.e. Round 31 offsets minus 0x10): state byte +1; if 0 return. Runs three parameter ramps (+0x30/+0x34/+0x38/+0x3c, +0x40.., +0x50..), calls the voice render (voice vtable +0x24), then by state: 2 = call vtable +0x18 (finished?), if true state := 0; 3 = call vtable +0x18, and when finished: if the pending note (s16 at +0x20) is negative, state := 0 (nothing to start); otherwise call the trigger fcn.42014b18(slot, note +0x20, previous note +0x22, velocity byte +0x24, mode 1, now); if the byte at +0x26 is non-zero afterwards, state := 2 and the voice release (vtable +0x14, argument byte +0x25) is called and the release stamp := now (a note-off arrived while the steal was pending); 1 = age counter +8 += 16. So the pending note starts in the same slot call in which the voice reports finished.
- Voice "finished" test (vtable +0x18, fcn.4204e87c): voice+0x530 == 6 (amp envelope stage 6).
- Fast fade (voice vtable +0x1c, fcn.42016280): if the release method (+0x14) is the default one (fcn.42015940) it is done inline: voice+0x14 := 0.0; the three envelope stage bytes at voice+0x510, +0x530, +0x550 := 5 (release); then in either case voice+0x52c (amp envelope Release value) := 0.01 (0x3c23d70a) and voice+0x5c8 := 1. The flag +0x5c8 stops the normal per-block control copy from overwriting +0x52c (cleared again by the next trigger, fcn.4037b61c). Release step index = trunc(0.01 x 127) = 1, and c[1] = 1.0 (ROM entry 1 is 32767), so at the next envelope stage update the amp level becomes 0 and the stage becomes 6. There is no exponential tail.
- Time to the fade: superseded by Round 45 (the update unit is the 49-sample control tick, not the 16-sample block).
- Round 8 statement about the envelope update rate: superseded by Round 45.
- Open (important): q = *(*(part+0x10)+0x50): resolved as far as the code allows in Round 45.

## Round 45: the part+0x10 pointer (q), the control tick, and envelope step timing (static; objdump of the whole image scanned, no emulator data used as a source)
- Control tick (part vtable +0x10, fcn.4037e71c, called once per 16-sample block): position part+0x4024 += 16; when it reaches the deadline part+0x4028 the tick counter part+0x402c += 1 and the deadline += 0x31 = 49 samples (plus a random -32..+30 samples only when q > 0.25). So the control tick is one per 49 samples on average (44.1 kHz: 900 Hz, 1.111 ms), at most one per block.
- Voices act only when their counter voice+0x5bc is behind part+0x402c (a new tick). The envelope stage machine (Rounds 8 and 44) therefore steps once per control tick when q < 0.5, and once per 8 ticks (392 samples, 8.9 ms) when q >= 0.5 (or on the first tick after a trigger). The step uses the same coefficient c[i] in both cases, so the envelope times differ by a factor 8 between the two. The earlier "once per 8 render blocks" text in Round 8 was wrong in its unit.
- q is read as *(*(part+0x10)+0x50) with a null check (null gives q = 0). The audio part constructor (fcn.42016a30) zeroes part+0x10, the init method (fcn.420169ec) writes only +4, +8, +0xc and +0x4068 (+0x4068 = engine + 0x10, the clock record), and the part base address literal 0x3fcbdcc4 occurs only in the getter and constructor. A scan of the whole image for stores at offset 16 through a pointer loaded from offset 4 or 8 finds no write in the engine, note processor or voice code (24 hits, all in unrelated list or UI code at 0x42008xxx to 0x4200afxx, 0x42010e3c, 0x420386ff, 0x42040329; not individually checked). So in v1.02 part+0x10 appears to stay null, q is 0, and the dormant hook does nothing: no per-voice random pitch offset, no tick jitter, and the stage machine steps on every control tick. If the pointer were wired to the engine parameter array (engine+0x32ec), q would be entry 0x14 (Flavour); nothing wires it, so Round 33 stands (no consumer). Unconfirmed: the scan is a heuristic and indirect stores would be missed.
- Envelope step coefficients (verified in the table builder at 0x420164d3 to 0x42016516): c[i] = ROM[i] x 0.0625 / 32767.0, or 1.0 when ROM[i] > 0x7ffe (entries 0 and 1). ROM table at 0x3c070340, 128 x u32 (values start 32767, 32767, 30500, 28500, 26500, 24800, 23100, 21600 and fall to 6, 6, 5).
- Time constants, assuming q = 0 (step every 49 samples, 44.1 kHz, one-pole step level += (target - level) x c, so tau = -T / ln(1 - c) with T = 1.111 ms). Index = trunc(value x 127). Index 0 and 1 are instant. Index 2: 18.5 ms, 5: 22.9 ms, 10: 32.7 ms, 20: 65.6 ms, 40: 264 ms, 60: 1.06 s, 80: 4.25 s, 100: 17.1 s, 110: 34.3 s, 120: 64.7 s, 127: 116.5 s. The same list multiplied by 8 applies if q >= 0.5. A release finishes when the level is below 0.0001, which takes about 9.2 time constants; an attack ends at about 0.996 (5.5 time constants); a decay runs until the level reaches the sustain level.
- Steal fade (Round 44) with q = 0: the amp level goes to 0 at the next control tick (at most 49 samples, 1.1 ms away), then the 16-sample gain ramp; roughly 0.4 to 1.5 ms. With q >= 0.5 it would be up to 8.9 ms plus the ramp.
- Consequence for the planned ms/seconds display: the display value for an envelope knob can be computed from the table above; the unit relies on the tick being 49 samples and on q being 0 in v1.02. Both are static findings and need a hardware check before the display is trusted.

## Round 46: app partition size (static; UF2 parsed and the app's partition loader read; emulator partition tables not used)
- UF2 (ref/PROTEIN_v102_original.UF2, sha256 prefix ffcd702affd22a88, identical to the attached copies): 2323 blocks of 256 payload bytes, family 0x12e6002f, flags 0x2000, target addresses contiguous from 0x0 to 0x91300 (594,688 bytes = the app image incl. checksum and SHA-256). Address 0 is relative to the app slot; the file gives no absolute flash address and no slot size. (Earlier text saying the original UF2 is not in the workspace is outdated: it is in ref/.)
- The app contains the standard ESP-IDF partition loader (code at 0x42030a20 to 0x42030b97; strings "No MD5 found in partition table", "Partition table MD5 mismatch"). It reads the table region at flash offset 0x8000 with length 0x1000 (literals 0x8000 and 0x9000), 32-byte entries with magic 0x50aa, MD5 marker 0xebeb. So the table sits at the IDF default offset 0x8000. Its entries are in the device flash only, not in the image.
- The app finds its two data partitions by name at run time: "data" (NVS) and "approm" (type 0x40, subtype 0, mapped read-only with the size taken from the table entry). No size is hard-coded for either, and the app never reads its own slot (no esp_ota_* or running-partition call).
- Image size 0x91300; the next 64 KB boundary is 0xA0000 (655,360 bytes), 60,672 bytes of headroom. The image header says 8 MB flash.
- Result: the real app slot size cannot be determined from the files available (UF2 and ELF). Known: slot >= 594,688 bytes. Assumed, not known: slot is a multiple of 64 KB, so >= 0xA0000. The planned growth of about 200 bytes fits under that assumption. Whether the Waldorf bootloader rejects an image by size is also unknown.
- To settle it: a read of flash 0x8000 to 0x9000 from a real unit (for example esptool read_flash 0x8000 0x1000, over the ESP32-S3 USB port in download mode) or the bootloader source. The partition tables in the emulator kit are guesses and are not evidence.

## Round 47 - Code classification (Protein vs library), static only

Method: function starts found by scanning the image for `entry a1,N` (2,402 candidates; the older r2 list of 2,611 had overlapping sizes, so the earlier "15% mapped" figure was wrong). Labels come from referenced strings, notes (exact function start), float-op density, then link-order region. Per-function table: analysis/function_map.csv. (Round 48 supersedes analysis/protein_unmapped.txt.)

Layout (confirmed by strings and float ops):
- IRAM 0x4037a000-0x4037e000 and a few later IRAM functions: Protein voice/render/tick. The rest of IRAM (0x40378000-0x4037a000 and 0x4037f990-0x4038d000): ESP-IDF (heap, spi, gdma, FreeRTOS, interrupts).
- IROM 0x42002dac-0x4200c000: ESP-IDF startup, panic, heap, uart, intr. Before it, 0x42000020-0x42002d00 is data.
- IROM 0x4200c000-0x4202c000: Protein application. Library pieces inside it by string: spi_master (0x4201597c), tinyusb_set_descriptors area (0x42016648).
- IROM 0x4202c000-0x42041000: ESP-IDF drivers (tinyusb, usb_phy, gpio, partition, mmu, wdt, spi, i2s, ledc, adc) and libm (0x4203c000-0x42040000: fcn.4203d800 and fcn.4203d928 are libm wrappers, so Round 42's "unverified: fcn.4203d800 is exp" is now: library math wrapper, exact function still unchecked).
- IROM 0x42041000-0x4204d990: newlib printf/strtod/dtoa plus tables.
- IROM 0x4204dd00-0x4204f978: Protein C++ vtable methods (many 8-byte stubs). 0x4204de44 and 0x4204e150 are code, not tables (corrected in Round 48: 16-sample block filters called from the EQ process fcn.42012058).
- IROM 0x4204f978-0x42053xxx: ESP-IDF (adc, gdma, sleep, panic, main task, tlsf).

Totals (code bytes, by function-to-next-start size): Protein 809 functions, 157,860 B; library 1,570 functions, 246,672 B; data or unclear 23 functions. Protein functions with their start in the notes: 319 functions, about 121,000 B (about 77% of Protein bytes). Not mapped: about 490 functions, about 37,000 B; 448 of them are under 200 B.

Confidence: library-by-string (211) and Protein-by-string/notes/float (about 370) are evidence-based. Region-only labels (about 1,360 library, about 440 Protein) rest on link order and are not checked one by one.

## Round 48 - Function map of the whole firmware (static, no emulator data)

Result: analysis/function_map.csv initially recorded 2402 entry-scan candidates. Current status after Round 90: 762 Protein functions (155808 bytes), all described (743 strong, 19 medium, 0 notes), 1638 library/platform functions, and 2 verified non-function data spans. The old Round 48 library-name counts are historical and may not reflect later reclassification. The Protein/library count differs from Round 47 because 50 small functions at 0x4037a000-0x4037a400 and 0x4200c05c-0x4200ca30 turned out to be ESP-IDF code, and fcn.4204de44 and fcn.4204e150 turned out to be code.

Use of the file: filter column `class` = P and `confidence` = low to see what to re-check before a patch. An entry is a reading of the disassembly, not a test on hardware.

Corrections made in this round:
- The earlier "15% mapped" figure came from overlapping sizes in the radare2 list; the own start scan gives 2,402 functions.
- fcn.42010af4 is not confirmed as the delay; the delay buffers (two of 0x1aeaa0 bytes) are made from Part__buildEffects through fcn.420114d8.
- fcn.4201cbfc is the UI constructor. fcn.4204f918 and fcn.4204f934 are SoundLibrary accessors. Some functions at 0x4037a0f4-0x4037a400 are ESP-IDF locks, not Protein code.
- Exact start of the effects setup is 0x4200f68c (an older note used 0x4200f68f, which is inside it).

Part__buildEffects 0x4200f68c (416 B): allocates effect memory with the capability wrapper fcn.403798c8 and flag 0x400 (the ESP-IDF value for external RAM; not checked in this build), logs "./components/utility/Memory.h" on failure. Two buffers of 0x1aeaa0 (1,764,000 B) at part+0x6e4 for the delay, two of 0xba1c (47,644 B) at part+0xa0 for fcn.42010af4, an object of 0x28d4 B, then the flanger (fcn.42012a20 at part+0x98c), phaser (fcn.42013d34), tremolo and others. 1,764,000 B is 441,000 samples of 4 B, that is 10 s at 44.1 kHz if the samples are floats (unconfirmed).

Wavetable cache layout, refined by the helpers 0x4202b7f0-0x4202b9f4 (cache object = arg1):
- Block address = cache + 4 + (table * 61 + slot) * 64. A table is 61 blocks of 64 bytes (3,904 B). Samples are 8 bit, offset by 0x80.
- Tables 0 to 88 (89 tables) occupy cache + 4 to cache + 0x54d44. Right after them, at cache + 0x54d44, is a 61-byte flag array per table (89 x 61), 1 = block is set (key or already built).
- Table 88 (cache + 0x53e04) is built by fcn.4202b7f0: block 0 is a triangle-like ramp (-126 up in steps of 3 for 31 samples, then down), block 1 is a constant -64, block 2 a descending ramp from 191. These are waves 61 to 63 of every table (Round 7).
- fcn.4202b884 builds one block from another with a 3-weight smoothing sum and sets its flag. fcn.4202b928 fills blocks 0 and 60 of a table with LFSR bits (feedback mask 0xffff8125) and smooths inward with 23 passes from each end. fcn.4202b9f4 builds a table from 61 records of 8 bytes. fcn.4202bb70 is the dispatcher over these.
- Consequence for the planned tool: a custom table needs 61 blocks of 64 bytes (3,904 B) or fewer key blocks plus flags 1 on the key slots, and the build path in 0x4202b6c4 / 0x4202c500 from earlier notes. The exact smoothing weights and the table-to-seed assignment are not decoded.

UI page classes (vtables installed by the UI constructor fcn.4201cbfc; the object offsets are not tabulated yet):
| vtable | class size | role (from its methods) |
|---|---|---|
| 0x3c073688 | 20 | generic page, input handler fcn.42021eac toggles byte this+10 on press |
| 0x3c0736d0 | 60 | Arp/Seq page, handler fcn.42022960 (Start/Stop recording, CC list actions) |
| 0x3c073758 | 40 | page, input handler fcn.42022cb4 (press/release with a mask) |
| 0x3c073788 | 224 | page with a std::function member and a sub-vtable 0x3c073854 |
| 0x3c073854 | 172 | Cancel/OK dialog (draw fcn.42024590, input fcn.420249a8), text of two 64-byte strings |
| 0x3c07380c | 16 | page with button handler fcn.42024440 |
| 0x3c0738ec | 12 | minimal page, no input |
| 0x3c073238 | 16 | page looking up a 120-entry table at 0x3fcce448+0xc60 |
| 0x3c0738c8 | n/a | modal opener (fcn.420249dc), three std::function members |
| 0x3c073210 | 20 | page redraw: clears two framebuffer regions |
| 0x3c0736ac | n/a | list page: draw fcn.42022120 (rows from y=8, 112 x 64 rect) |

Known gaps: the 1,405 unnamed library functions; the 132 low-confidence entries; the UI offset table; table-to-seed assignment in fcn.4202bb70; the smoothing weights in fcn.4202b884.

Round 48 addendum - UI object offsets read from fcn.4201cbfc and the UI helpers (offsets are from the UI object start; the vtable of each object was not matched to its offset except where stated):
- ui+0x2c: sub-object built by fcn.42021aa8 (default page, sub-vtable 0x3c0736ac, default label "[Off]"); also the page that fcn.4201d11c makes current.
- ui+0x198: FrontPanel__ctor_WaveFilterAmpMaster (fcn.4201e108), with a second part at ui+0x9a4.
- ui+0x548: note-on consumer (fcn.42022bc0); ui+0x584: fcn.4204f0f8; ui+0x79c: Map-CC learn hook (fcn.4201dee4). All three get every incoming MIDI message from fcn.4201d4b4.
- ui+0x5ac: fcn.420237f8; ui+0x75c: fcn.42024138; ui+0x7ac: fcn.42024574 (Cancel/OK dialog class, 172 B).
- ui+0x12a8: 1,024-byte framebuffer (128 x 64, 1 bit per pixel); ui+0x16a8: popup text area, flag ui+0x16bc; ui+0x1200+164 (ui+0x12a4): redraw flag set by page changes and popups; ui+0x1700+204: MIDI log flag. std::function members at ui+0x17d0, +0x17e0, +0x17f0.
- Static numbers worth keeping for the build: BPM = 20 + 380 x ("%.0f", fcn.42016ae0); StepLen table at 0x3c073118 (4 to 1/16 in beats); swing s = 0.05 + 0.9 x; arp start/next-note selectors fcn.4201799c, fcn.42017900, fcn.42017a20 with the held-key object (flags bytes 0..127, 64-bit press times at +0x280); held-key order modes 0..6.

Round 48 addendum - effect memory and object sizes (from Part__buildEffects fcn.4200f68c and the deleting destructors; allocation flag 0x400 is the ESP-IDF external-RAM capability, not checked in this build):
| effect | object offset | object size | buffers made at setup |
|---|---|---|---|
| Chorus | part+0xa0 | 0x204 | 2 x 0xba1c (47,644 B), attach fcn.42010af4 |
| Compressor | part+0x2a4 | 0x440 | none (init fcn.4200f87c) |
| Delay | part+0x6e4 | 0x10c | 2 x 0x1aeaa0 (1,764,000 B), attach fcn.420114d8 |
| Flanger | part+0x98c | 0xb4 | 2 x 0x28d4 (10,452 B), fcn.42012a20 |
| Drive / EQ / Phaser | part+0x7f0 / +0x83c / +0xa40 | not read | none |
| Tremolo | part+0x13fc | not read | none |
| Reverb (probable) | part+0xfd0 | not read | 1 x 0x40040 (262,208 B), fcn.42013758 |
Master EQ low shelf: fcn.4202a308 at part+0x5c. The EQ process fcn.42012058 calls two 16-sample block filters, fcn.4204de44 and fcn.4204e150 (Round 47 listed them as data by mistake). Chorus and delay resets sit right before their process functions (fcn.4200fe48 before fcn.4200feb4, fcn.42010b2c before fcn.42010ba0).

Round 48 addendum - held-key object layout (relevant to the latch change; read from fcn.420170dc, fcn.4201728c, fcn.4204ebd8, fcn.4204e918, fcn.4204e93c):
- bytes 0x000-0x07f: held flag per note (non-zero = held);
- floats 0x080-0x27f: velocity per note (4 B each, address 0x80 + 4 x note);
- 64-bit values 0x280-0x67f: press time per note (8 B each, address 0x280 + 8 x note);
- the arp reads the object through a pointer at arp+0 and keeps the current note at arp+8; the order mode is the word at arp+4 (0..6) in fcn.42017900 and fcn.42017a20, and the starting-note mode in fcn.4201799c.
- Consequence for latch: the order modes by press time and by velocity depend on the per-note time and velocity entries, so a latched note should keep its time and velocity fields (do not clear only the flag in a way that leaves stale times when it is re-pressed); this is an inference from the readers, not tested.
- This layout agrees with the noteOn writes noted earlier (flags at obj+0xa0, velocity floats at obj+0x120, times at obj+0x320), so the held-key object starts at obj+0xa0 of the note processor and the order readers use exactly the fields that noteOn fills.

Round 48 addendum - note-on and note-off of the note processor, read in full (fcn.42014528 and fcn.42013e90; relevant to latch):
- Held flags are only set by noteOn and only cleared by noteOff (and by the 128-byte clear in noteOn when ChordMode switches on). The byte obj+0x1be8 (F232) is the active ChordMode copied from the parameter byte obj+0x1be9 (F233) at each note-on (fcn.420143bc uses F232 as the mode: 1 = chord table 0x3fcce170 with type index at obj+0x1b00+244, 2 and above = scale table 0x3fcce098 with scale index at +245).
- noteOff clears held[note] only when F232 is 0. When ChordMode is on (F232 non-zero) a single note-off does nothing to the held table; the chord notes are released together when the chord root is released (chord list at obj+0x1bea, count at obj+0x1b00+242, root at +243).
- Voice noteOn/noteOff (slots +20 and +24 of the voice owner) are only called when the arp/seq is not running (byte obj+0x1b5c is 0); with the arp running the held table alone drives the arp.
- Latch consequence (inference, not tested): the smallest change is in noteOff, skipping the held[note] clear when a new latch flag is set and byte obj+0x1b5c is non-zero; noteOn then needs the "first key after all keys were released clears the table" rule, for which no physical-key record exists today (the held table is the only record). A separate 128-bit "physically down" bitmap would be needed, or a counter of keys down.
- With ChordMode on, single note-offs do not clear the held table (read from the code). Whether and where the old chord is cleared when a new chord root is played was not read (fcn.420143bc and the loop after 0x42014610 were only skimmed). Not confirmed on a unit.

Round 48 addendum - arp start/stop and the held-table check (fcn.4201763c, fcn.42017534, fcn.42017570, fcn.420175d8):
- fcn.42017570 runs each tick: it scans 32 words of the held table; non-zero and runner idle starts the runner (fcn.42018810, runner object at obj+0x13d0); all zero and runner active sends all-notes-off and re-initialises the note slot table (fcn.4204eda0). So the arp already stops by itself when the held table becomes empty and restarts when it fills; a latch that keeps held flags set keeps the arp running without another change here.
- Start (fcn.4201763c) sets the flag word obj+0x1434 and binds the runner by mode (fcn.420175d8: requested mode obj+0x142c to applied mode obj+0x1430); stop (fcn.42017534) clears the flag, stops the runner, sends all-notes-off and clears the 128-byte table at obj+0x1438.
- Not read: the runner itself (fcn.42018810, fcn.4201883c) and the generator step fcn.42017b0c was mapped in an earlier round.

Round 48 addendum - wavetable builders, finished (read from fcn.4202b884, fcn.4202b928, fcn.4202b9f4, fcn.4202bb70):
- Smoothing in fcn.4202b884 is a (1, 6, 1) / 8 low-pass with +4 rounding on the unsigned bytes; the two edge neighbours are the bitwise inverse of the end bytes.
- Tables 71, 72, 73 ('K+St. #1' to '#3') are Karplus-Strong tables: seeded noise in blocks 0 and 60, smoothed 23 times inward from each end (key blocks 0-23 and 37-60), seeds 74, 0x5678 and 113 (written to cache+2 before fcn.4202b928).
- Tables 74 ('from1to5') and 75 ('19 / 20') are built from 61 records of 8 bytes (data at 0x3fca08f4 and 0x3fca070c): three partials per record (amplitude shift and phase step) from a 184-entry sine table at 0x3c078c4c, plus an optional LFSR noise part.
- The switch in fcn.4202bb70 covers tables 28-31 and 56-75; the cases for 28-31 and 56-70 were not read, so how those waves are made (sync sweeps, PWM, fuzz) is still open.
- For the custom wavetable tool: a user table needs only the 61 blocks of 64 bytes (offset binary, 0x80 = zero) and a key flag of 1 on the blocks that are real; unflagged blocks are interpolated by the build step in buildTable (fcn.4202b6c4, earlier rounds); this last point is an inference, not read.

Round 48 addendum - UI page types by offset (vtable stores in fcn.4201cbfc matched to class sizes and to the page-draw table above; the offsets of the first six pages follow from the constructor calls, the rest from the vtable store addresses and the sizes adding up with no gaps):
| ui offset | size | vtable | page (draw method) |
|---|---|---|---|
| 0x02c | 140 | 0x3c073634 | sound list (fcn.420216d0, fcn.420217b0) |
| 0x0b8 | 224 | 0x3c073788 | save page, name editor (fcn.42023088) |
| 0x198 | about 0x39c | several (0x3c0732ec ...) | front panel sections, fcn.4201e108 (second part at ui+0x9a4) |
| 0x534 | 20 | 0x3c073688 | arp / play page (fcn.42021d00: 'Play', 'Note', 'Octs') |
| 0x548 | 60 | 0x3c0736d0 | sequencer page (fcn.4202223c; note-on consumer fcn.42022bc0) |
| 0x584 | 40 | 0x3c073758 | value readout page (fcn.42022d98) |
| 0x5ac | 432 | 0x3c0737ac | layer pages (fcn.420234ac, fcn.420236ac) |
| 0x75c | 36 | 0x3c0737dc | header 'Multi Mode' / 'Single Mode' (fcn.42024090) |
| 0x780 | 16 | 0x3c07380c | settings-style page with button handler fcn.42024440 (draw fcn.4202415c); role inferred from its handlers |
| 0x790 | 12 | 0x3c0738ec | boot / version screen (fcn.42026198) |
| 0x79c | 16 | 0x3c073238 | CC mappings list (fcn.4201dbf0; learn hook fcn.4201dee4) |
| 0x7ac | 204 | 0x3c073830 | debug lines '0: %.1f < %.1f' (fcn.420244c8) |
| 0x878 | about 0x130 | 0x3c0738c8 | mapping view '->' (fcn.42024b88) and modal opener fcn.420249dc; callback fcn.4201d274 at ui+0x988 |
| 0x98c | 20 | 0x3c073210 | page with redraw fcn.4201cbb8 |
Offset plus size equals the next offset along 0x2c, 0xb8, 0x198 and along 0x534, 0x548, 0x584, 0x5ac, 0x75c, 0x780, 0x790, 0x79c, 0x7ac, which supports the pairing (0x780 is 0x75c + 36). The object between 0x878 and 0x9a4 and the front panel part from 0x198 were not split further.

Round 48 addendum - rhythm clock and arp runner fields (read in full from fcn.42017e50, 42017ec8, 42017f78, 42017ff8, 42018098, 42018104, 420181e8, 42018260, 42016b28, 42016b40, 4204ed80, 42018810, 4201883c):
Rhythm clock object (a sub-object of the note processor; defaults from fcn.42017e50):
| offset | type | meaning |
|---|---|---|
| +0 | pointer | owner record: 64-bit clock time at +48/+52, a float at +4 and a flag byte at +80 (names not established) |
| +8/+12 | double | next step time |
| +16/+20 | double | gate-off time (1e100 = none pending) |
| +24/+28 | double | last step time |
| +32/+36 | double | current step length |
| +40/+44 | word pair | start time copy |
| +48 | byte | current step active |
| +49 | byte | copy of owner+80 |
| +52 | float | gate (relative), default 0.5 |
| +56 | float | StepLen index 0-16, default 10 (0.25 beat) |
| +60 | float | swing amount, default 0.5 (straight) |
| +64 | float | default 1.0 (read by fcn.42017f78; role not established) |
| +68 | int | rhythm pattern index 0-15 |
| +72, +76 | pointers | modulation inputs for gate and swing |
| +80/+84 | double | step length in beats (from the table at 0x3c073118) |
| +88 | int | step counter |
| +92 | table | 16 patterns x 32 steps x 8 bytes {u8: 0 rest, 1 note, 2 tie; float level 1.0} (0x100 per pattern, ends at +0x105c) |
| +0x1000+92, +96, +100 | ints | current pattern, pattern length (32), step index |
Rhythm patterns 1 to 15 (16 steps, repeated twice in the 32): 1 x.xxx.xxx.xxx.xx, 2 x.x.x..xx.x.x..x, 3 x.x.x.xxx.x.x.xx, 4 x.xxx.x.x.xxx.x., 5 x.x.xx.xx.x.xx.x, 6 xx.x.xx.xx.x.xx., 7 x.x.x.x.xx.x.x.x, 8 x.x.x.xx.x.xx.x., 9 xxx.xxx.xxx.xxx., 10 xx.xx.xx.xx.xxx., 11 xx.xx.xx.xx.x.x., 12 xx.xx.x.xx.xx.x., 13 x.x.x.x.xx.x.xxx, 14 x..x..x..x..x..x, 15 x.x.x.x.x..xx.x. ; pattern 0 is all steps on. Byte value 2 (tie) is not produced by this table; where it is set was not searched.
Per step: fcn.42018104 (advance) fires a step when the clock passes +8/+12 and the pattern byte is 1; fcn.42018260 (gateOffDue) later fires the gate-off at +16/+20 unless the byte is 2; fcn.420181e8 sets the gate-off time as last step time + gate x length.
Arp runner (object at obj+0x13d0): +0 byte bound flag, +0xc generator object, +0x10 rhythm clock, +0x14 byte, +0x18/+0x1c two pointers copied into clock +0x48/+0x4c (gate and swing modulation inputs), +0x54 state (0 running, 2 stopped), +0x58 word cleared at start.
Internal tempo object (fcn.42016b40): bpm = 20 + 380 x (double) at +8 and +16, samples per beat = 44100 x 60 / bpm = 2,646,000 / bpm at +24 (CORRECTED in Round 50: the constant 0x41442ff8 is the high word of the double 2,646,000.0, not 2.5e6; the 2,500,000 figure belongs only to the MIDI clock input above), time at +48/+52; its state struct (fcn.42016b28) has x = 0.263 (120 BPM) at [0].

### Round 48 addendum: display, event queue and UI helpers (last low-confidence pass)

All 462 Round 48 entries now have confidence strong, medium or pattern; none are left at low. Pattern means the behaviour was only partly decoded.

- Framebuffer: 128 x 64 one bit, column-major, byte index = x*8 + (y>>3), bit = y&7 (1,024 bytes, ui+0x12a8). Helpers: 0x4204f6a0 hLine, 0x4204f720 rectOutline (vertical edge loops partly inferred), 0x4204f868 clearRect, 0x42028b60 string draw through the font vtable.
- Event queue (global 0x3fcd3298, static init 0x42028a58): ring of 1,024 entries of 16 bytes {a, b, type, 0}, buffer 0x4000 bytes, head +16, count +20, capacity +8, entries at *(+4). Entries start as {-1, 0, -1, 0}. Push sites log "CircularBuffer::write: is full!" when full. Types seen: 7 (parameter change, 0x4201993c), 9 (position, 0x420196f8), 13 (full dump marker, 0x4201a848).
- Parameter values travel as fixed point x 1,000,000 (event type 7: {param id, trunc(value * 1e6), 7, 0}; receiving side multiplies by 1e-6, 0x4202020c). The send-all loop 0x4201a848 runs ids 0 .. 0x230, so the parameter id space has 561 entries (0x231, from the loop bound).
- Front panel page 0x4202019c reads parameter id 54 + 84*v (v = signed 7-bit byte at ui+0x900+160). What v selects was not traced.
- CC map walker 0x4201db90: s16 table at 0x3fcce448+12+2*(0x630+n), 120 entries, -1 = unmapped.
- UI messages: two forms exist. 16-byte ring events {a, b, type, 0} via fcn.4201d010 (types 8, 17, 20, 24 seen in callbacks) and 24-byte messages to the std::function at ui+0x17e0 (type 12 in fcn.4201d03c, type 1 with value 100 in 0x420220d0).
- Layer set (sound engine part): four layers at this+8, +0xcc0, +0x1978, +0x2630 (stride 0xcb8); byte at layer+0x3c8 is a flag whose change to non-zero clears 0x600 bytes at layer+0x6ac (0x4204e844).
- Sound edit buffer global 0x3fccf428: name at +0x8c4 (24 bytes), cleared by calling Sound__setName with an empty source string (0x3c060e4c; "No Name" follows at 0x3c060e50).
- LED and timing: 0x420271d4 square-wave blink (period u16 at +6), 0x420272f4 20 ms accept check, 0x4202713c pattern stepper (field meanings inferred).

Still open (not requested): cases for tables 28-31 and 56-70 in fcn.4202bb70, where pattern byte 2 (tie) is written, which front panel sections use 0x420209b8 and 0x42021070, meaning of the signed byte at ui+0x900+160, boot animation role of 0x4201915c (inferred only).

## Round 49: generated wavetables (fcn.4202bb70) and the tie flag

Resolution of the range conflict: the earlier note "0x1c-0x4b except the user range" and the later "28-31 and 56-75" are the same set (28-31, then 56-75 after skipping the User tables 32-55). 24 tables are built by code at boot. The other 65 tables (0-27 and 76-87 from the wave ROM, 32-55 empty User slots, 88 by fcn.4202b7f0) are not covered here. Read from the disassembly only; nothing was run, nothing was checked on hardware. The table names agree with what the code builds (names are at 0x3fca05ac).

Common layout facts, checked by arithmetic against every literal in the function: table t starts at cache + 4 + t*3904 (wave w at + w*64), its 61 valid flags start at cache + 0x54d44 + t*61. Every loop below runs exactly 61 waves. Stored bytes are offset binary (0x80 = zero); only the first half cycle is stored (see the playback notes).

ROM data blocks used (all in rodata just below the wave ROM):
- 0x3c078f40, 2048 bytes: 1024 signed 16-bit sine values (value i = round(32767 sin(2 pi i / 1024)), verified at 12 points). The wave ROM starts right after it at 0x3c079740.
- 0x3c078f00, 64 bytes: half-sine bump (128, 178, 219, 246, 255, ...).
- 0x3c078d04 (64 bytes, "A"), 0x3c078e80 (about 60 bytes, "B"), 0x3c078e00 (125 bytes), 0x3c078ec0-0x3c078efc (61 bytes), 0x3c078c0c (64 bytes): irregular byte runs that do not look like designed waves (for example B starts 72, 67, 210, 69, 109, 32, 255, 81). They look like arbitrary data reused as raw material for the "Fuzz" tables. What they originally were is not known.

Per table (case in the dispatcher):

| Table | Name | How it is built |
|---|---|---|
| 28 | MW Sync | Phase accumulator (16 bit, not reset between waves). Sample = 128 + ((phase >> 8) & 63). Increment per sample 280 + 32 w (w = 0..60). A 6-bit sawtooth with rising frequency. |
| 29 | MicroPWM | Wave w: samples 0 .. 63-w are 0xA0, the rest 0x80 (a positive pulse that shrinks from 64 to 4 samples). |
| 30 | Glassy | Only waves 0 and 60 are written and flagged: wave 0 = the 64-byte half-sine at 0x3c078f00, wave 60 = the same bytes ANDed with 0xC0 (2 bit). The 59 waves between are filled by the interpolation routine. |
| 31 | SquareHP | Three flagged waves: 30 = falling ramp 0xC0 down to 0x81 (byte = 0xC0 - k); 60 = flat 0x80, then from about sample 47 rising exponentially to 0xFF (a running value doubles every second sample, byte = (value >> 24) OR 0x80); 0 = 0xFF - (wave60 AND 0x7F). Rest interpolated. The wave 60 formula is read from branch targets, partly inferred. |
| 56, 57, 58 | SawSync1-3 | Sawtooth: byte = (phase >> 16) low 8 bits, phase starts at inc/2, inc starts 0x40000 and rises per wave by 0x1111 / 0x3333 / 0x7777 to 0x8110d / 0x103327 / 0x20775b, so the frequency multiple sweeps 1x-2x / 1x-4x / 1x-8x. The last 16 samples are faded to the centre ((byte-128) * (63-i) / 16). |
| 59, 60, 61 | PulSync1-3 | Same as 56-58 with a square: byte = 0xFF when bit 23 of the phase is set, else 0x00. inc starts 0x20000, step 0x199a / 0x3bbc / 0x8000, end 0x819b2 / 0x103bcc / 0x208000 (1x-4x / 1x-8x / 1x-16x). Same 16-sample fade. |
| 62, 63, 64 | SinSync1-3 | Sine from the 1024-entry table: index = (phase >> 6) & 1023, byte = (sine >> 8) XOR 0x80. Increment per sample starts 535 / 1047 / 2071 and rises per wave by 64 / 128 / 256 to 4439 / 8855 / 17687 (512 = one half cycle per 64 samples). Same 16-sample fade. |
| 65 | PWMPulse | Wave w (a = 59-w): samples 0..a are 0xFF, the rest 0x00. |
| 66 | PWM Saw | Wave w: starts at 128, rises by 2 per sample for the first 60-w samples, then by 2w+8 per sample (8 bit wrap). |
| 67 | FuzzWave | Wave w = one-pole low-pass (acc = (acc + x) >> 1, acc reset to 0 per wave) over 64 ROM bytes taken from 0x3c078e00 + w, then XOR 0x80. |
| 68 | Distortd | Wave k: samples 0 .. k+3 from block A (0x3c078d04 + i), the remaining samples from block B (0x3c078e80 + i - k - 4). A splice whose head grows by one sample per wave. |
| 69 | HeavyFuz | A 16-bit shift register (seed 200, feedback constant 43, state kept in the first two bytes of the cache) gives a stream of 0x01 / 0xFF values; each following wave is the previous one scrolled by one sample with one new value at the end. Read quickly; details unconfirmed. |
| 70 | FuzzSync | Wave 0 = 64 bytes copied from 0x3c078c0c. Waves 1..60: byte 0 = 72, then n = 62-w bytes from 0x3c078ec0, then the same n bytes in reverse order and inverted (NOT), starting at offset n. Only the first 2n bytes are written, so the end of the later waves is not written by this code (it stays at the zero the cache was cleared to unless something else fills it; not checked). |
| 71-75 | K+St. #1-3, from1to5, 19 / 20 | Already documented (Karplus-Strong seeds 74, 0x5678, 113 and the record-built tables). |

Consequence for custom tables: all of these 24 are plain code with data in rodata; replacing one means either keeping its case but overwriting the cache after it (call buildTable-style fill with your own 61 x 64 bytes) or editing the dispatcher so that table number does nothing and the data is copied from your own block. Both are build-phase design, not started.

### Rhythm pattern tie flag (byte 2)

- The pattern table (clock+92, 16 patterns x 32 steps x 8 bytes) is written by only two functions found: the clock default init fcn.42017e50 (every step 1) and the pattern loader fcn.42017ec8, which fills patterns 1-15 from the ROM table at 0x3c072d54 (15 rows of 16 words, 64 bytes per row, repeated over 32 steps) and stores only 1 (word non-zero) or 0 (word zero).
- A scan of all Protein code for a byte store of a register set to the constant 2 found 11 sites, none of them a pattern write (the nearest, fcn.420174f0, marks released notes at obj+0x1438+note, which is the note-state array also used by the sustain pedal).
- So in v1.02 the value 2 (tie) is read by fcn.42018104 and fcn.42018260 but I found nothing that can ever write it. This is a negative result from a static scan: stores of a computed value, wider stores, or a block copy into the table would not have shown up. Unconfirmed.


## Round 50: tempo clock, rhythm clock constants and sync chase (static, not run)

Corrections to earlier readings:
- "7.17" is not a float. 0x40e58880 is the high word of the double 44100.0 (sample rate), stored at TempoClock +64/+68 by fcn.42016b40 (low word 0) and also used by the Phaser init (0x420132bc, same high word).
- "0.0442" was a miscalculation. The double 0x3fa58106_20000000 used by the rhythm clock reset functions is 0.042 (0.04199999943).
- 0x41442ff8 (high word) is the double 2,646,000.0 = 44100 x 60.

ROM routines used here (from the IDF ROM linker script): 0x40002184 __adddf3, 0x40002250 __divdf3, 0x400022a4 __extendsfdf2, 0x400022d4 __fixdfsi, 0x40002334 __floatsidf, 0x40002418 __muldf3, 0x4000246c __nedf2, 0x400024fc __subdf3, 0x4000252c __truncdfsf2. fcn.4203d964 is newlib ceilf (checked from its code); fcn.4203da78 is its pair, probably floorf (not read).

TempoClock object (note processor obj+0x10; first 16 bytes are the TempoState set by fcn.42016b28):
- +0 float x (BPM parameter, setParam id 5). +4 float, default 1.0 (TempoState[4]): used only as a switch, "enabled when above 0.1", by Time__update and the rhythm clock reset. No setParam id writes obj+0x14, so it stays 1.0 in the code I searched (not exhaustive).
- +8/+12 double BPM, +16/+20 previous BPM, +24/+28 double samples per beat (44100 x 60 / bpm), +32 byte "tempo changed" flag (triggers a "ChutneyLog" line), +48/+52 double beat position, +56/+60 double chase target, +64/+68 double 44100.0, +72/+76 double offset (a whole number of beats), +80 byte "locked to external position" flag, +88/+92 double last Song Position handled (init -1.0).
- Time__update (fcn.42016bb0, a3 = obj+0x70): a3 is a copy of the transport record built by fcn.4201c074 (copied by fcn.4204e628, vtable 0x3c070110 slot at 0x3c070168): +0 byte (external sync mode), +1 byte (tempo valid), +8 double tempo, +16 byte and +24 double (beat position valid and value), +32 byte and +40 double (Song Position valid and value).
  - If owner+4 > 0.1 and a3[1] is set, BPM = a3 tempo, else BPM = 20 + 380 x. Samples per beat is recomputed every block.
  - On a new valid Song Position (a3[32]), the position is re-aligned: new = ceil(current position) + fractional part of the Song Position; the offset (new minus SPP) is stored at +72, +80 is set to 1, +88 remembers the SPP, a log line is written.
  - While a3[0] and a3[16] are set the position chases the external beat position: target = a3 beat position + offset, position += 0.04 x (target - position) per block (a gain of 0.04, from the double 0x3fa47ae1_40000000). If the external flags drop, +80 is cleared.
- Rhythm clock reset (fcn.42017f78 and its twin 0x42017f08): copies owner +48/+52 to the clock's +40 and +8 (next step time). When owner+4 > 0.1, owner+80 is set and the clock float at +64 (init 1.0, only ever read, no writer found) is above 0.1, the first step time becomes ceilf(position - 0.042): the next whole beat, with 0.042 beats of grace (about 21 ms at 120 BPM) so a start just after a beat still plays that beat.

### Round 50: front panel sections, current part, overlay and init task

- Front panel object (constructor fcn.4201e108): section objects and their vtables - +0x10 Detune (handler fcn.4202049c, vt 0x3c073400), +0x40 Chord (fcn.4202020c, vt 0x3c0733d0), +0x70 Wave (fcn.4201f6a0, vt 0x3c073310), +0xa0 Wave envelope knob group (vt 0x3c07325c), +0xd0 Filter (fcn.4201fd4c, vt 0x3c073340), +0xf8 Filter envelope knob group (vt 0x3c07328c), +0x128 Amp (fcn.4202006c, vt 0x3c073370), +0x14c Pan (fcn.42020140, vt 0x3c0733a0; draw fcn.4202019c), +0x170 Amp envelope knob group (vt 0x3c0732bc), +0x1a0 LFO (fcn.420209b8, vt 0x3c073430), +0x1d8 FX (fcn.42021070, vt 0x3c073460), +0x234 Master (fcn.42021420). This matches KNOBS.md; the three handlers that were listed as "section not matched" are Chord, LFO and FX.
- The signed byte at ui+0x900+160 (= ui+0x9a0) is the current part. Evidence: the pot writer notes add part x 0x54 to UI indices below 0x150 and name ui+0x9a0 as the part; fcn.4201d03c stores it and posts message type 12; fcn.4202019c reads parameter 54 + 84 x part (RandPan of that part). Which values it takes (0..3 for four parts) is assumed from "4 parts" in the engine notes.
- fcn.4201cee8 is called only from ParamValueOverlay__draw (0x4201c9c4) and opens the page at ui+0x98c, which is therefore the parameter value overlay shown when a knob is turned (inferred from the call site).
- fcn.4201915c is the "Init task" (name string confirmed at its creation in the boot splash fcn.420192cc, stack 0x1000, priority 4).

## Round 51: pattern-rated Protein functions raised, demo player, page handlers
All Protein functions that were rated "pattern" are now decoded from disassembly (aligned branch targets where the linear listing desynced). No Protein function is left at pattern level; 189 are rated medium (role inferred), the rest strong or from the notes. Still static only, nothing run.
- Sequencer step advance 0x42018500 (renamed SeqStep__advancePosition; it is not an arp function): n = min(32, int(SeqLen + 0.05) + 1 + int(31 x mod + 0.05)); n < 2 gives position 0; SeqDir 1 backward, 2 ping-pong without repeating ends, 3 ping-pong with repeated ends, 4 forward once then stop (position byte 0xFF), otherwise forward with wrap. Fields: SeqDir +0x28c, SeqLen +0x28d, modulation pointer +0x294, position +0x299, direction +0x29b.
- Boot demo player (fcn.4201c280 start, fcn.4201c358 tick, fcn.4201c2e0 switch entry; the earlier "Seq__toggleEntryEvent" name was wrong). Table at 0x3fca04bc, stride 12: pointer to a Waldorf SysEx sound dump (F0 3E 2F ...; first starts "Sound Love", second "Sgiv"), length 0x1550, base note 60, step 3, time byte (112 and 16) and note count (65 and 4). The start/switch code sends the dump through fcn.4201a848 (SysEx import, then push of all 561 parameters), then due time = sample counter (note processor +0x38, 64-bit) + 4.0 x the double at note processor +0x28. The tick injects MIDI note-on (0x90, velocity 100) and note-off (0x80) through fcn.42019bdc; note += 3, minus 12 above 71, so notes cycle 60, 63, 66, 69; after the last note the due time moves by 0x15888 (88,200) and the other entry is loaded. This corrects the earlier note that it plays 4 notes only: entry 0 plays 65 notes, entry 1 plays 4. Unconfirmed: the time unit of the time byte and of the double at +0x28 (assumed samples). This is a boot-time effect candidate for disabling.
- UI pages identified from the page table: fcn.42021eac and fcn.4202202c belong to the Play/arp note page (state 4, vtable 0x3c073688): note 0..108, octaves 0..4, note-off/note-on messages through the std::function at ui+0x17e0; fcn.42022cb4 is the sequencer step page input (state 6): step 0..31, pitch parameter id 0x1a9 + 4 x step, velocity id 0x1aa + 4 x step; fcn.42022120 draws a generic 3-row text list with a scroll thumb; fcn.420249dc is the value edit modal (modes 0 to 3, callbacks at +184, +200, +216; class size 232); fcn.420248e8 is the on-screen keyboard press handler ('<' is backspace).
- Front panel section ring (fcn.4201e4f4): Wave, Filter, Amp, Wave env, Filter env, Amp env, Detune, LFO, FX, wrap to Wave; group 1 shows Detune or (shift) Chord; groups 3 and 4 LFO; group 5 FX.
- fcn.420282b0 is a SysEx sender (length-prefixed block) that packs bytes into 4-byte USB-MIDI style event packets for port 0 (port records at 0x3fcd6660, stride 480); the earlier name Flash__readHeaderRecords was wrong. That port 0 is the USB port is inferred from the packet format.
- The 215 destructors, forwarders, setters, constant returns and no-op methods were each checked against their disassembly; descriptions matched. fcn.4204e664 corrected (calls inner slot +0x50 and clears 128 bytes at this+0xa0). The row at 0x42010b02 is not a function (interior of fcn.42010af4).
- Open from this round: time units of the demo player timing; what the demo player is for; which controls send group numbers 0 to 5 to the front panel handler; the caller of fcn.420282b0.

## Round 52: medium-rated arp, held-key and clock functions (first batch)
- ArpSeq__start / stop / checkHeldStart / bindMode (0x4201763c, 0x42017534, 0x42017570, 0x420175d8) confirmed from aligned code. Bytes at obj+0x1400: +52 enabled, +53 runner active, +54 set to 1 when the runner is started by a held key. The arp stops by itself the moment all 32 held words are zero (all-notes-off, runner stopped, slot table reset); only a change that keeps the held flags set would let it keep running (relevant to the latch idea, nothing changed).
- HeldKeys next-note selectors (0x42017900 forward, 0x42017a20 backward): case numbers 0 to 6 confirmed; the backward version swaps each pair of cases (0/1, 2/3, 4/5). Case 1 of the backward version was not aligned in the decode.
- Correction: RhythmClock__swingStepLength (0x42017ff8) only visibly computes s = 0.05 + 0.9 x (clamped); the earlier statement that odd steps use (1 - s) is not seen in this function and is now marked unconfirmed.
- Not yet re-checked in this round: NoteProcessor__noteOn / noteOff, RhythmClock__advance / setGateOff / gateOffDue, ArpSeqPage__onEvent, and the other medium rows.

## Round 52 addendum: note processor, rhythm clock and record page re-checked
- NoteProcessor__noteOn / noteOff (0x42014528, 0x42013e90) checked against the disassembly: the engine calls (note-on slot +20, note-off slot +24) go to the engine object at obj+8, not obj+0. Switching ChordMode on runs the reset (virtual slot +80; the default stub fcn.4204e5f4 does engine slot +0x50 then clears the 128 held flags). Old chord root is released through this object's own note-off.
- RhythmClock__advance (0x42018104): clock time is rounded through a float before the compare; the active byte is 1 only when the pattern byte is exactly 1; the step index is advanced after the read. RhythmClock__gateOffDue (0x42018260) therefore tests the NEXT step's pattern byte: a tie (2) in the next step suppresses the gate-off. setGateOff confirmed. (Tie value 2 is still never written by any code found, so ties do not occur in v1.02.)
- 0x42022960 is the record page event handler (state 5, ui+0x548), not a generic Arp/Seq page: item lists per mode (Off 1, Arp 9, Seq 13), actions -9 start recording and -10 open step editor, UI id = parameter id + 0x19b (0x19b ArpSeqMode, 0x19d ArpOctaves, 0x1a0 BPM, 0x1a4 SeqLen), BPM edited in whole BPM with the law 20 + 380 x.
- Task done for the six functions named in the last request; the remaining medium rows (about 180) are not re-checked.

## Round 53: small medium-rated functions checked against the disassembly
- 107 medium-rated functions of 130 bytes or less were compared with their disassembly. 79 are now rated strong (descriptions matched). Strong count 370, medium 96, pattern 0, low 0.
- Corrections found: fcn.42012a58 resets FOUR sub-objects in each of its two arrays (not eight); fcn.42014eac works on the layer sub-objects at this+8, +0xcc0, +0x1978, +0x2630 (earlier text had +0xcc8, +0x1980, +0x2638); fcn.42016fcc draws a second random value in -1..+1 (mod 20001 x 1e-4 - 1.0) stored at this+16, then clears 27 records of 8 bytes from this+0x12c; fcn.4201e338 is a debounced section switch (this+0x260 = current section pointer, changed only if 200 ms passed since the stamp at this+0x268; page 2 is shown when 1000 ms have passed since the stamp at this+0x26c), so the front panel handler fcn.4201e4f4 group 0 acts only while this+0x260 is zero; fcn.4202746c uses different calls for pin 45 and for the direction call (identities inferred).
- Held-key scans fcn.420170dc / fcn.4204ead0 / fcn.420171bc / fcn.4201728c: structure confirmed (scan of bytes 0..127, 64-bit compare of times at +0x280/+0x284, -1 when the note is above 127 or nothing is held); the exact tie-break order was not simulated, so they stay medium.
- Still medium (96): mostly the larger functions - wavetable builders, layer page input, the four sysex functions, EQ block filters, UI constructor, the LFO/FX/Chord section handlers, tempo clock update, parameter overlay, chord builder, board test screens, reverb core setParam, dialogs, event and parameter push functions, held-key scans, and several constructors with truncated descriptions.


## Round 54: chord table, chord builder, wavetable tables 31/69/70/88 (static, not run)

Chord table (fcn.42016e78, vtable 0x3c082e24 static init). Replayed the stores. Chord table at 0x3fcce170, 17 entries of 36 bytes (up to 4 semitone offsets, count at +32): 0 {0,4,7}, 1 {0,3,7}, 2 {0,5,7}, 3 {0,3,6}, 4 {0,4,8}, 5 {0,5,10}, 6 {0,4,7,11}, 7 {0,3,7,10}, 8 {0,4,7,10}, 9 {0,5,7,10}, 10 {0,4,7,9}, 11 {0,3,6,10}, 12 {0,3,6,9}, 13 {0,4,8,10}, 14 {0,4,8,11}, 15 {0,4,6,10}, 16 {0,3,7,11}. Scale-degree table at 0x3fcce098, 6 entries: 0 {0,2,4}, 1 {0,2,4,6}, 2 {0,3,4}, 3 {0,3,4,6}, 4 {0,4,8}, 5 {0,2,4,6,8}. Chord names are inferred from the intervals only.

Chord builder (fcn.420143bc): F232 = 0 nothing; F232 = 1 root plus table offsets, folded into -128..127; F232 >= 2 root snapped up to the scale mask at obj+0x1bf8 (stride 8), then each note = snapped root moved up by the table's number of scale steps (next non-zero mask entry, +12 on wrap). The scale branch is not folded into 0..127.

Table 88 (fcn.4202b7f0): block 0 = 0x82, 31 steps of +3, last value repeated, 31 steps of -3; block 1 = 64 x 0xC0; block 2 = 0xC0 then 191 down to 129; all 61 flags set.

Corrections and closures in fcn.4202bb70 (all cases now read from aligned disassembly):
- Table 31 SquareHP wave 60 is exact: 0x80 up to sample 48, then 0x81,0x81,0x83,0x83,0x87,0x87,0x8F,0x8F,0x9F,0x9F,0xBF,0xBF,0xFF,0xFF,0xFF for samples 49-63. Wave 30 = 0xC0 - j. Wave 0 = 0xFF - (wave 60 & 0x7F). Flags set for waves 0, 30, 60 (other waves interpolated).
- Table 69 HeavyFuz confirmed: LFSR seed 200, step = shift left, if bit 15 shifts out then xor 43 and toggle the value between 0x01 and 0xFF; wave 0 byte 0 = 1 and bytes 1-63 from 63 steps; each later wave = previous bytes 1-63 moved down by one plus a new last byte (previous last byte, toggled when the LFSR fires). The state is kept at cache+0 (16 bit). All 61 waves flagged.
- Table 70 FuzzSync correction: the earlier statement that the end of later waves is never written is wrong. Wave w (1-60) is a pattern repeated with period 124 - 2w (starts at every multiple of the period up to byte 63): byte 0 = 72, then src[1..61-w] from 0x3c078ec0, then NOT of src[61-w] down to src[0]. Spill of a wave into the next is overwritten by the next wave's own pass; the last repeat of wave 60 ends exactly at byte 63, so nothing spills into table 71. Flags are set for waves 1-60. Unconfirmed: the wave 0 flag (0x55df2) is not set by this code; the ROM copy at 0x400011f4 is assumed to return its destination (consistent with the flag pointer starting at wave 1).
- Consequence for custom tables: each code-built table writes its 61 x 64 wave bytes and sets flag bytes at cache + 0x54d44 + t*61 (key waves only for tables 30 and 31, all waves for most others; unflagged waves are interpolated at build time, inference). A replacement therefore needs the wave bytes plus the flag bytes. Build-phase design, not started.


## Round 55: SysEx output messages and CC output (static, not run)

All outgoing messages start F0 3E 2F, device id (settings value 3, not 8 as an earlier description said), a message type byte, and for the three dump types a running sequence byte (counter byte at 0x3fccff50, 7 bit). Every message ends with a checksum byte (sum of all bytes after F0, masked to 7 bits) and F7. A 32-bit word is always sent as five 7-bit bytes (bits 31-28, 27-21, 20-14, 13-7, 6-0). Messages are built in a 1024-byte buffer and queued by Product__sendMidiSysexMessage (object 0x3fcce448); the last argument of each sender is the output port.

| Type | Sender | Content |
|---|---|---|
| 0x0F | sendResponse (0x4201b050) | request command byte (this+0), response code (two 7-bit bytes), 32-bit value, 15 bytes in all. Codes from callers: 1 slot load failed, 2 wrong slot, 3 wrong checksum. |
| 0x10 | dumpSound (0x4201ad58) | 6 messages: total count 6, number 1-6; message 1 adds 2 raw bytes from sound+0x8dd, +0x8dc and the 24-char name; then 98 parameters per message (5 tag bytes + 5 value bytes each, tag from fcn.42018b3c, value from fcn.4204f8dc), 561 in all, last message 71. Sizes 1016 and 990 match the embedded factory sound messages; the embedded last message is 480 bytes (47 parameters), 24 fewer than now, so the embedded sounds are an older layout (inference). |
| 0x21 | sendPresetNames (0x4201aa34) | total messages, message number, start slot (two 7-bit bytes), then up to 31 slots of 32 bytes: 24 name chars, used byte, 7 zero bytes (all zero for an unused slot). |
| 0x31 | dumpScreen (0x4201b7dc) | 2 messages (count 2, part 1 or 2), each 128 little-endian words of the display buffer 0x3fcd1214-0x3fcd1614 (512 bytes per message). |

The request handler for a preset dump (0x4201b1dc): resets a stack Sound object, loads the slot with fcn.42029a78, then dumpSound, or sendResponse code 1.

CC output (0x42019f44, renamed Param__sendAsCc): param-to-CC byte table at controller+0xd5c (negative = none); sends {0xB0, cc, value}. Value rules: in-part parameters 5 and 12 (Wtb1Pos, Wtb2Pos) = step/(steps-1)*127; stepped parameters with at most 32 steps = (step*128+64)/steps, with more steps = step + 64 - (steps-1)/2; bipolar = 64 + round((x-0.5)*126); else x*127 rounded; all clamped 0..127. Helpers ParamInfo__isBipolar/isStepped/stepCount (0x42018e80, 0x42018f08, 0x42018f9c) read the 28-byte per-part descriptor (table 0x3fc9fb78, entries by index mod 84; second table 0x3fc9e2dc for 0x150-0x230) and the 32-byte type record at 0x3c06f5c4 (+8 step count, +24 bipolar flag). Their earlier descriptions were wrong (copied from an unrelated notes line). Unconfirmed: the caller of Param__sendAsCc, and the exact route through the note processor singleton in the three helpers.


## Round 56: front panel section handlers chord, LFO, FX (static, not run)

Handler signature (all section objects): this, a3 control id, a4 value, a5 event kind, a7 flag. Kinds: 0 button, 2 pot, 3 encoder, 7 engine-to-UI parameter update (the chord description in Round 48 said kind 3 for the update, wrong). The owner byte at [this+4] byte 0 is the Shift flag. Section state is kept in the section object; changes are written by fcn.4201ea04 (pots), fcn.4201eda0 (encoders) and FrontPanel__stepButtonWriter (button toggles). Parameter indices below are the per-part UI indices (add current part * 84).

Chord section (0x4202020c): +36 chord mode (set by kind 7 for 0x22b), +40 selected field 0-2, +44 field-select mode (toggled by button 10). Pot 0 with Shift: ChordType 0x22c (0x22d when mode is 2 or more). Encoder 1 with Shift: ScaleType 0x22e. Encoder 0 with Shift: in field-select mode moves the field (mode 1 toggles 0/2, mode 0 stays 0, else clamp 0-2); otherwise edits field 0 = ChordMode 0x22b, 1 = ScaleType 0x22e, 2 = ChordType 0x22c / 0x22d.

LFO section (0x420209b8) also holds the mod matrix: +36 state (0 idle, 1 field select, 2 mod matrix), +40 field 0-2, +48 mod slot 0-7, +52 LFO select 0/1. Pot 10 = LFO speed (Shift clear, writes Clocked 0) or ratio (Shift set, writes Clocked 1); pot 11 = mod amount of the slot (index 60 + 3n); encoder 3 = mod destination (59 + 3n); encoder 4 = mod source (58 + 3n) or, with Shift, LFO shape (46 / 51). Encoder 0 changes LFO select, field, or mod slot depending on state; button 10 enters field select and then toggles Clocked (45 / 50) or Sync (47 / 52), or returns. Changing the LFO select sends a type 11 message to the std::function at ui+0x17e0 (purpose unconfirmed).

FX section (0x42021070): +36 slot (FX 1 or FX 2), +40 / +44 effect types (0-9, set by kind 7 for 0x150 / 0x151), +48 + 4*type selected parameter within the type, +88 alt flag (button 10). Pot 12 sends the parameter: Shift set = list entry at 0x3c073488[type][selected], Shift clear = 0x3c0735d4[type]. Encoder 0: slot select, or (alt flag set) edit the effect type; encoder 5: Shift set = choose parameter within the type (count from the list at 0x3c0735ac[type]), Shift clear = edit the type.

Relevance to the planned envelope remap: pots 10 and 11 share the LFO section with the mod matrix, so a remap that re-uses these pots must keep the mod matrix path working.


## Round 57: layer page, parameter overlay, UI constructor (static, not run)

Layer page (fcn.420239d4, page at ui+0x5ac): UI parameter 0x229 LayerMode is both the layer selector (0-3) and the Multi flag (4). Buttons 0-3 select layer n (LayerMode = n) outside Multi; in Multi they select the part and, if it was already the current part, toggle that layer's ActiveInMulti (parameter 83 + 84n). Button 4 enters Multi, or restores the remembered layer (+37) when the Multi/Single header page is current. The encoder moves between five rows when row-select mode is on (Destination param 56, Active, Volume param 55 in steps of 0.01, PolyMode param 57, Actions); button 10 acts on the row or toggles row-select mode on the Volume row. The Actions row opens a list: Init Layer, Load Layer, Copy Layer, Paste Layer (only when the clipboard flag at +44 is set), Close. The sound-select popup (slot at +0x1ac, 360 slots) uses button 10 once to confirm the slot (3000 ms timeout) and again to apply it (SoundSelect__apply).

Parameter overlay: ParamOverlay__trigger (fcn.4201cee8) opens the overlay page ui+0x98c only after the same control has changed value for more than 199 ms (consecutive changes under 400 ms apart) and the UI is idle; it stays 400 ms. ParamValueOverlay__draw (fcn.4201c9c4) shows the parameter name, the formatted value and a 120 x 4 pixel bar at x = 4, y offset + 18 (unipolar left to right, bipolar from the centre at x = 64, stepped = step / (steps - 1)). The clear of the byte at this+13 in the trigger was not found.

UI constructor (fcn.4201cbfc) page offsets: ui+0x2c sound list, +0xb8 and +0xc4 pages (vtables 0x3c073788, 0x3c073854), +0x198 front panel, +0x534 play, +0x548 record, +0x584 step, +0x5ac layer, +0x75c multi/single header, +0x780 settings, +0x790 (0x3c0738ec), +0x7ac debug, +0x878 modal (vtable 0x3c0738c8, ValueEditModal), +0x964 list helper, +0x98c parameter overlay, +0x9a4 edit sound / parameter store. The 1024-byte display buffer is at ui+0x12a8 = 0x3fcd1214, the same buffer dumpScreen sends, which confirms the SysEx screen dump start address (0x3fccff6c + 0x12a8).


## Round 58: record wavetable builder, event pushes, held-key selectors (static, not run)

Record-built tables 74 and 75 (fcn.4202b9f4) are not stored as full waves: each wave is three sine partials (sine table of 184 signed bytes at 0x3c078c4c, shifts 0-7, phase steps) written into samples 0-22 only, then samples 33-63 are set to the negation of samples 30-0 (antisymmetric), and samples 23-32 stay at 0x80 unless the optional noise part (record bytes 6 and 7) is added over all 64 samples. The noise is a held random +/- amplitude step (amplitude from the 17-byte table at 0x3c078bf8, update rate from record byte 7) from a 16-bit shift register restarted at 0xFFD5 per record. No flag bytes are written: the interpolation routine fcn.4202b5ac does nothing when the table's wave 0 flag is clear, which also explains table 70 (wave 0 flag clear, waves 1-60 built directly). Closes the Round 54 open item about the table 70 wave 0 flag.

Controller object (0x3fcce448): sound/parameter store at +0xfe0 (0x3fccf428), SysEx queue 0x3fcce438. ParamEvent__push sends {6, index, value} to the engine and a UI event {index, value x 1e6, type 7}. Controller__tick pushes the pending preset load as UI event type 9 and runs the boot demo player (state byte +0xfdd). Event__DUMP_RANGE dumps slots a3..a4 with MidiSysexController__dumpSound, port 1 when setting 133 is 1.

Held-key selectors (relevant to the latch work). The held-key object has held flags at +0, velocity floats at +0x80 and 64-bit press times at +0x280. Starting note (fcn.4201799c, mode word at this+4): 0 newest, 1 oldest, 2 highest note, 3 lowest note, 4 highest velocity, 5 lowest velocity, 6 newest (not a reset, correcting an earlier note). Forward and backward next-note selectors (fcn.42017900, fcn.42017a20): modes 0/1 step by press time, 2/3 by note number, 4/5 by velocity, 6 resets to -1; none wraps around (they return -1 at the end; that the arp then restarts from the starting-note picker is an inference). A latch must therefore keep the held flags, velocities and press times intact, as stated in Round 52.

## Round 59: effects setup, reverb parameters, EQ filter (static, not run)

Part__buildEffects (0x4200f68c) is called once from engine init 0x4200ce38 on the effects object at engine+0x188. It gets the tempo clock pointer from fcn.4201467c (clock+16 and clock+0x728 are passed on) and allocates with heap_caps_malloc (0x403798c8), caps 0x400. That 0x400 means external RAM (MALLOC_CAP_SPIRAM) comes from ESP-IDF header knowledge, not from this binary. A failed allocation only logs and carries on.

- Delay at part+0x6e4: 2 x 1,764,000 B, cleared with ROM memset, stored at +100 and +136, tempo clock at +252. Delay__reset (0x42010b2c) is virtual slot +12 and is inlined in Delay__attach.
- Chorus at part+0xa0: 2 x 47,644 B, stored at +216 and +252 (Chorus__attachBuffer), cleared; Chorus__reset sets gain 1.0 at +0x1f8 and the flags.
- Flanger buffers at part+0x98c: 2 x 10,452 B (0x42012a20, not re-read here).
- Compressor at part+0x2a4: Compressor__init stores +92 (result of fcn.4203d85c(0.1, 1.0/0.0), a power-like helper, not confirmed), +96 = 0.99948, byte +128 = 1, +0x378 = 40, +0x370 = (+0x374 + 40) mod 92. The use of the 92-slot ring is not confirmed.
- Stub inits: Drive +0x7f0, EQ +0x83c, Phaser +0xa40. Tremolo at +0x13fc stores the two clock pointers at +132 and +136.
- Reverb at part+0xfd0: one 262,208 B buffer, cleared inside fcn.4202acc8 on the sub-object at this+12.
- Last call fcn.4202a308(part+92, 0.00527, 4.56) (master EQ low shelf per earlier notes, not re-checked).
- Total requested: 3,906,400 B.

Reverb core setter (fcn.4202a824, index 0..7, raw value kept at this+0x3d0+4*idx), correcting Round 48:
- 0: this+0x1c = x*0.5.
- 1: v = 16370*x*(1+x) as int, min 0x7fe0, multiple of 16, plus u16 at +8 plus 16, stored as u16 at +10.
- 2: clamp 0..1 into +0x24.
- 3: stored only.
- 4: s = 2x-1. s <= 0: s clamped to -1, A = -0.99 - 1.89*s, B = C = (1-A)/2. s > 0: t = min(0.3 s^2, 1), A = 0.99 - 1.69*t, B = (A+1)/2, C = -B. Written to +0x3c4/+0x3c8/+0x3cc and to each 16-byte tap record from this+0x208 (count at +0x200).
- 5: per 36-byte record (count at +0x16c) float at +0x188+36k = float at +0x180+36k * 20 x^3.
- 6: float at +0x17c+36k = float at +0x174+36k * 5x.
- 7: +0x14 = 2x, +0x10 = 2-2x.
What the fields mean (size, decay, diffusion) is not confirmed.

EQ filters (0x4204de44 and 0x4204e150) are the same direct form I biquad over 16 floats in place: y = b0 x + b1 x1 + b2 x2 + a1 y1 + a2 y2, coefficients at this+4 (a1), +8 (a2), +12 (b0), +16 (b1), +20 (b2) shared, states +24/+28/+32/+36 for the first and +40/+44/+48/+52 for the second (verified: identical instruction streams once the state offsets are mapped). Which band or channel each state set serves is not confirmed. The value formatter 0x4200f9e4 prints x*87/44100*1000 as milliseconds (0 to 1.97 ms). 0x42014fcc (Part__noteOffToLayers) sends the voice manager note-off to layer a5 when the float at part+0x3328 truncates to 3, otherwise to every active layer matching a6 (-1 = all); the identity of that float is not confirmed.

## Round 60: voice manager and sub-object constructors (static, not run)

Voice lookup: voice object = 0x3fcaaf78 + i*0x828 (fcn.420162bc) with i = part index * 8 + slot number, so 8 voices per part; part object = 0x3fcbdcc4 + i*0x4074 (fcn.42016a20).

VoiceManager layout (vtable 0x3c0701a8): +4 part pointer, +8 a word set by someone else (0 at construction), 8 slots of 0x70 bytes from +0x10, +0x390 u16 (0x100 after reset), +0x392 voice count 8, +0x394 word 0x64003c (mono/legato flag byte at +0x397), +0x398 byte, +0x3a0/+0x3a4 stamp pair, pointer table +0x3a8 to six 16-byte records at +0x3cc, +0x3ec, +0x3dc, +0x3fc, +0x40c, +0x41c (word +12 holds 0xa0000 or 0x640000 at construction, the low u16 is cleared by reset), a 128-byte area at +0x62c, three 512-byte areas at +0x6ac, +0x8ac, +0xaac and one at +0x42c. This corrects Round 48 (four 32-byte records). The use of the records and areas is not confirmed.

VoiceManager__reset (a3 = part index): clears the above, calls part virtual slot +8 with (part, part index, manager, word +8), runs VoiceSlot__start for the 8 slots (the third argument is the slot number, not a note), then sets every parameter k = 0..82 through its own setParam to the word at +20 of the descriptor returned for k (probably the default, not confirmed). VoiceSlot__start stores the slot number, the voice pointer (+0x2c), manager word +8 (+0x28) and three area pointers (+0x60/+0x64/+0x68), clears its flags, and calls the voice's virtual slot +12 with the part pointer, the pointer table and the three 16-byte blocks at slot +0x30/+0x40/+0x50. What the voice does with them is not read here.

Constructors: RandomSource (vtable 0x3c071a68) holds a 35-record {word, byte} table with four self pointers (record 0, 32, 33, 34 to this+4, +12, +16, +8) and a 27-record table, then draws a random float into +12. ParamMgr builds the arp object at this+20, the chord and scale object at this+84 and the clock at this+0x368 (Round 48 said 0x744 for the arp; that is not what the code passes). ChordScale holds 32 records {u16 0x3000, 0.8, 1.0}, 0xf00 at +0x28c, 12 records of 8 bytes at +0x29c and the scale cache byte -1 at +0x304. Bits__fieldGet returns six 2-bit fields: 0 = byte1 bits 0-1, 1 = byte3 bits 2-3, 2 = byte3 bits 5-6, 3 = byte0 bits 5-6, 4 = byte0 bits 0-1, 5 = byte1 bits 5-6; only caller found is Board__scanInputs.

## Round 61: confirmation of the Round 60 open items (static, not run)

Corrections to Round 60 and 59, all read from code:
- The class at voice+0x5e0 (vtable 0x3c071a68, 0x248 bytes, last member of the 0x828-byte voice) is the modulation matrix, not a random source: its 35 records {float pointer, flag} are the ModmatSrc choices (names at 0x3c071ae0) and its 27 records are the ModmatDst slots (names at 0x3c071a74). Renamed ModMatrix__ctor, ModMatrix__setSource (0x4204e8c0), ModMatrix__setDestFlag (0x4204e8d0).
- Voice__init (0x4201597c, voice vtable slot +12, 1804 bytes) is what the voice does with the pointers it gets: it stores them at voice+0x5b8, +0x5cc, +0x5d0, +0x5d4, +0x5d8, +0x5dc and binds every modulation source id to a float cell: 1 to 3 envelopes (voice+0x4f8, +0x538, +0x518), 4 to 9 LFO cells (+0x578, +0x598, +0x5a0, +0x594, +0x59c, +0x5a4), 10 Mod Wheel, 11 Pitch Bend, 13 Channel Pressure, 16 Pedal, 17 Expression, 18 Breath to the six manager records, 12 Note PB, 14 Poly AT, 15 MPE Y to the slot ramp cells, 19 to 28 CC 22 to CC 31 to the CC value array, 29 Keytrack, 30 Velocity, 31 Rel Velo to voice+24, +16, +20. Ids 0 and 32 to 34 (Off, Random, Rand Bipol, Constant) are the ModMatrix's own cells. Destination flags: 1 for Pitch, O1 Pch, O2 Pch, else 0 (what the flag changes is not read). First call only: it builds an 18-float table at 0x3fcbb478 and a 128-row table (32-byte rows, divided by their sums) near 0x3fcbc4f0 with ROM double math; the formulas are not decoded.
- The six 16-byte manager records are linear ramps for part-wide controllers: {current, target, step, u16 ticks left, u16 length}. 0x3cc Pitch Bend (length 10), 0x3dc Channel Pressure, 0x3ec CC1, 0x3fc CC4, 0x40c CC11, 0x41c CC2 (length 100). Writers: VoiceManager__pitchBend 0x42014788, VoiceManager__channelPressure 0x420147f4, VoiceManager__controlChange 0x4201486c. The slot function fcn.4037a794 steps the per-voice ramp cells at slot+0x30, +0x40, +0x50 towards the per-note arrays (slot+0x60, +0x64, +0x68).
- The 512-byte areas are 128 floats each: manager+0x42c holds the last value/127 of every CC number, +0x6ac poly aftertouch per note, +0x8ac note pitch bend per note, +0xaac CC74 per note. +0x62c is the 128-byte note state table of the sustain logic.
- Descriptor +20 is the default value. Evidence: ParamMgr__loadDefaults (0x4201584c) writes it into all 149 global parameters, VoiceManager__reset writes it into the 83 voice parameters, the UI Init Sound action (confirm dialog, then the loop at 0x42023f97 with the popup "Initialized!") pushes the +16 field of the parameter-type record for the 83 per-part parameters, and +16 equals +20 in all 75 type records and all 83 voice descriptor rows. Voice descriptors: 32 bytes each from 0x3c072274, +0 name, +4 short name, +8 step count, +12 choice list, +16 and +20 default, +24 bipolar flag, +28 hook.
- loadDefaults also shows the engine object layout: managers at +8 + 0xcb8*i, 149 global floats at +0x32ec, four 0x14c-byte records (83 floats if the size is the hint) at +0x3540 + 0x14c*i whose pointers become manager word +8.
- Round 59 open item solved: the float at engine+0x3328 is global parameter 15, LayerAlloc (Layered, Round Robin, Random Robin, Midi Split); value 3 = Midi Split, which routes the note-off to one layer. Global parameter 14 is LayerMode (Solo A to D, Multi).
- Bits__fieldGet is the encoder A/B reader: six 2-bit fields of the 4-byte input shift register image (Board+0x12a8), one per encoder, fed by Board__scanInputs to Encoder__decodeStep (0x4202713c, the earlier name LedPattern__step was wrong): standard quadrature table at 0x3c0739d0, partial counts, a full step at config byte +7, and a step multiplier chosen by the time since the last step. Event pushed: {encoder index, delta, type 3, 0}. Knob names per index (Select, Semitone/Scale, Table, Target, Source/Shape, Type/Param) are the Round 19 photo match, not code.
Still open here: the two first-call tables in Voice__init, the ModMatrix evaluator (reading of the destination records and flags), which of the two step multipliers (config bytes +2 and +6) is larger.

## Round 62: confirmation of the Round 61 open items (static, not run)

Read from code:
- Voice__init (0x4201597c) first-call tables (guard: float at 0x3fcbb478 equal to 0). Table 1, 0x3fcbb478: 18 floats, entry i = 0.00059725868 / D[i], D[i] = note length in quarter notes from the 18 doubles at 0x3c0702b0 (16, 12, 8, 4, 3, 2, 1.5, 1, 0.75, 2/3, 0.5, 0.375, 1/3, 0.25, 1/6, 0.125, 1/12, 1/16); one per LFO ratio choice (4/1 to 1/64); the LFO rate code is tempo x table[ratio]. Table 2, 0x3fcbb4c0: 128 rows of 8 floats, taps at x = -3 - r/128 + j, sinc (cutoff constant 0.1764) times Kaiser window (beta 2, half width 4), each row divided by its sum. Table 3, 0x3fcbc4c0: 128 rows of 12 floats, x = -5 - r/128 + j, same sinc, Kaiser beta 9, half width 6, rows normalised to sum 1. The voice render uses them as fractional-delay interpolation filters (phase bits 17 to 23 pick the row, bits 24 to 31 the start sample, 128-sample ring). Unconfirmed: the unit of the constant 0.00059725868.
- ModMatrix__evaluate (0x4037e8cc, wrongly classed as library before): 8 rows at +0x204 (source id, destination id, float amount at +4). Destination ids 0 and above 26 and source id 0 or a null source pointer are skipped. x = amount x 2 - 1; if the destination flag byte (record +4) is non-zero x = x x |x|; accumulator += source value x x, clamped to -100..+100. The flag is set only for destinations 3, 4, 5 (Pitch, O1 Pch, O2 Pch), so the flag means a curved amount. Not checked: whether a destination no row touches is cleared elsewhere (the evaluator keeps its old value).
- Encoder numbers, from the section handlers (the printed panel names are not in the binary): encoder 0 = section select/edit; 1 = oscillator semitone (UI 0 / 7); 2 = wavetable (UI 4 / 11); 3 = modulation destination; 4 = modulation source / LFO shape; 5 = FX value. The match with the earlier photo names (Select, Semitone/Scale, Table, Target, Source/Shape, Type/Param) is by function, so it is consistent with them but the label text itself is not code evidence. The oscPitch and wave handlers were read only in part.
- Layer blocks: Engine__setParam (0x420157e0) stores every id at this+0x32ec+4*id (481 floats: 149 global, then 4 x 83), and for ids 149 and up calls the layer manager with k = (id-149) mod 83; the voices read the block at voice+0x5b8 at byte offset 4*k. So the four 0x14c-byte records are the 83 sound parameters (0x14c = 83 x 4).
- Correction to Round 61: VoiceManager__setParam (0x4037a430) does not store the value, so VoiceManager__reset's loop over k = 0..82 with descriptor +20 does not load layer defaults (only k = 57 uses the value, as the mono flag). ParamMgr__loadDefaults sets global ids 0..148 only. Descriptor +20 as the default value still holds for the global parameters (and the BPM descriptor 0x3e86bca2 matches TempoState__init). How layer sound parameter defaults reach the float block at boot is not found (likely a loaded sound).

Still unconfirmed: the unit of the LFO table constant; relative size of encoder step multipliers (config bytes +2 and +6); the rest of the oscPitch and wave handlers; ChordScale 32 x 20 block meaning; use of ModMatrix flags beyond the evaluator.

Housekeeping: analysis/function_map.csv was damaged by a faulty script and rebuilt from the base classification, the saved description files and the notes. The descriptions are complete. The confidence column lost the in-place upgrades of Round 53 and later (now strong 349, medium 126, low 10; before: strong 429, medium 51, low 0), so it understates; the Protein count is 764 (earlier 761 plus the corrected ModMatrix row), and 'mapped (notes)' rows quote a notes line that may differ from the earlier one.

## Round 63: UI task, UI tick and board helper functions (static, not run)

Read from the disassembly; 36 map entries (map119, map120), all rated strong except the console report.
- UI task (fcn.4201c6a0 creates it, fcn.4201c56c is its body). Correction: 0x3fccff68 is the task-handle output, the task argument is 0; priority 1, stack 0x1000, core 0 as stack argument (create-task call identified by argument shape). The body first switches to the default page, then installs three std::function senders in the UI object: ui+0x17d0 posts to queue 0x3fccff60 (200-tick timeout, logs on failure), ui+0x17e0 to queue 0x3fccff54 (the 24-byte engine event queue), ui+0x17f0 to queue 0x3fccff58. Then it loops: items from queue 0x3fccff64 (4 words) go to the UI event dispatcher fcn.4201d680; items from queue 0x3fccff5c (3 words, the MIDI bytes) go to UI__broadcastMidi; then the service step fcn.4201d498; then a one-tick delay.
- UI tick (fcn.4201d3ec): redraws when the dirty byte is set or 1000 ms have passed, but never more often than every 50 ms; clears the 1,024-byte framebuffer, calls the current page's draw slot (+12), draws the info popup or the overlay object at ui+0x17bc, the list overlay at ui+0x964, then sends the framebuffer to the display (fcn.420273f0). Correction: the popup's non-flag branch calls the object at ui+0x17bc, not the current page.
- Page switch fcn.4201d11c is fully traced: it makes the default page (ui+0x2c) current and calls the old page's leave slot (+24) and the new page's enter slot (+20), unless the current page is the one at ui+0x7ac.
- Correction: fcn.4201d498 does not run a wavetable cache constructor. It calls fcn.42025074 every loop, which is a debug console and statistics report acting once per second (core load, free stack of the UI, Engine (Main), Helper and ExtraEngine tasks, heap info, sizeof lines for SynthVoice, SynthTimbre and dsp::MW1WavetableCache). Read from its strings and call pattern only (medium).
- Storage__logStats (fcn.42027be0) does not open a partition: it logs NVS entry counts for partition "data" and, with a flag, lists namespace, key and type of every entry. The NVS callees are identified from argument shapes and log text. fcn.42033380 was classed as data and is code (now library).
- Board helpers: GPIO 46 and GPIO 6 are driven from flags, GPIO 15 and 16 from two bits (register addresses from the ESP32-S3 register map). Board__gpioSetup resets pins 48, 38, 47, 46 and 45 and sets drive and direction as listed in its entry. What these pins are wired to is not known from code.
- Not yet read in this group: the dialog show variants, the settings, sound list, save and layer page handlers, the board test screens, the font and framebuffer drawing functions.

## Round 64: sound list, save, layer, header, settings and debug pages (static, not run)

22 map entries (map121), all read in full from the disassembly and rated strong.
- Overlay and message mechanism: fcn.4201d328(ui, object, ms) sets a modal overlay object (word ui+0x17bc, deadline ui+0x16b8, active byte ui+0x16b4); with object 0 it removes it. fcn.4201d2dc(ui, text, ms) copies a 256-byte message into ui+0x16bc. The first byte of that buffer is the "popup set" test used by the UI tick and fcn.4201d36c. This corrects the Round 63 wording that called ui+0x16bc a flag byte.
- Sound list page: Select button (button 10) loads the highlighted slot (UI event type 9 with the slot number; an unused slot shows "Empty" for 2 s); encoder 0 steps through the 360 slots, saturating at 0 and 359, optionally filtered by the slot used flag (filter byte +96, 255 = all). The roles of two flag arguments of the encoder handler are not known from the function.
- Layer page: a layer menu callback runs Initialize (item 0), Copy (2) and Paste (3, only when the paste buffer byte +44 is set); item 1 leaves layer-select mode. The slot step saturates at 0 and 359 (the earlier text said wraps); the layer number steps modulo 4. Applying a slot posts UI event type 10 with value layer x 4 + a byte from ui+0x9a0.
- Settings page: 134 settings (0..133); Select on a setting whose range is below 4 steps it with wrap-around, otherwise toggles value edit; the encoder either edits or scrolls through visible settings. Side effects: Transpose, Master tune and MPE status post engine events 7, 8 and 9; display and LED brightness call the display and LED objects; index 10 writes a byte at 0x3fcce448. Correction: the button dispatcher requires its second argument to be non-zero (earlier text said zero).
- Corrected from Round 53 and 57 wording: SoundSelect "wrap at 360" is saturation; Select does not "draw a value bar".
- Not yet read: dialog show variants, board test screens, font and framebuffer drawing, held-key and voice-manager helpers, the front panel section handlers read in part (wave, oscPitch), console report fcn.42025074 in detail.

## Round 65: dialogs, text entry, fonts, drawing, boot LED task, board test screens (static, not run)

18 map entries (map122, map123), all read in full and rated strong.
- Value edit modal show methods: fcn.42024de4, fcn.42024e7c, fcn.42024f28, fcn.42024fcc are the show methods for modes 0, 1, 2 and 3 of the value edit modal (Round 51 mode table matches): each copies one or two 64-byte texts to +10 and +74, stores the mode (+148), cursor (+152), navigate flag (+156), values and bounds (+160 to +172), swaps its std::function callback into +184 (mode 0), +200 (modes 1 and 3) or +216 (mode 2), then makes itself the overlay for 0x47868c00 ms (effectively until closed). Mode 3 also stores two words at +176 and +180 (probably a label list and count; unconfirmed).
- Correction: fcn.42024590 is the text entry draw (title at the word +148, line at y 22, edited string at y 26 with a blinking cursor, buttons "Ok" at x 6 and "Cancel" at x 70, y 48), not a Cancel/OK value dialog. fcn.420248e8 was re-read and stands.
- Fonts: two fonts, each a vtable with height (slot +8), width (+12), column bits (+16), spacing (+20): font 1 (0x3c073b40) is 16 rows with spacing 1 and 16-bit columns; font 2 (0x3c07423c) is 24 rows with spacing 2 and 32-bit columns. Both cover characters 32 to 128. Font__drawString stops when a glyph would pass the right limit and returns the x after the last glyph.
- Drawing: the line function is the integer Bresenham algorithm; the rectangle outline is fully decoded (bottom row and right column are not clipped on the other axis).
- Boot LED task: a template of ten LED ids (1, 3, 5, 7, 9 to 14) is stepped every 150 ticks, one LED lit at a time, for as long as the flag byte 0x3fca04b8 (initial value 1 in the image) is non-zero; the controller tick (fcn.420196f8) clears that byte on every call, which ends the animation, and the task then logs "initTask DONE!" and deletes itself. The earlier text said 150 ms; the tick length is not known from the binary.
- Board tests: audio test plays 293.3 Hz (left) and 440.0 Hz (right) at 0.308 of full scale if the sample rate is 44.1 kHz (the rate comes from the earlier note, not from this function); the button and encoder tests count presses of all 16 buttons and both turn directions of all 6 encoders. All three finish when the word at board+0x124 is non-zero (EDIT button; inferred from the prompts).
- fcn.4203db20 was classed as data and is a sine-like libm function (now library).
- Still medium after this round: the held-key and voice-manager helpers, the front panel wave and oscPitch handlers, effect and layer broadcast helpers, static initialisers, descriptor accessors, and the console report.

Addendum to Round 65 (demo player, static): the boot demo player (fcn.4201c280 start, fcn.4201c358 tick, fcn.4201c2e0 switch) lives in the controller at +0x1af8 and is driven by Controller__tick: state byte controller+0xfdd = 1 waits for the ms time to reach the word at controller+0x1b04, then starts the player and sets state 2; state 2 calls the player tick on every controller tick. The player note-on and note-off messages are {0x90 or 0x80, note, 100} sent through the MIDI router with fifth argument 2. Midi__injectNoteOff and UI__postReceiving were re-read and stand. Unconfirmed: the time unit of the due time and the double at note processor +0x28.

## Round 66 - Play and Record pages

- PlayPage__onInput (0x42021eac), PlayPage__stepOctave (0x4202202c) and PlayPage__noteOff (0x420220d0) re-read and confirmed. LED 13 is on only when the current page is the page at ui+0x548.
- RecordPage__start (0x420228c0): logs "Start Record", clears step counter +0x36, sets recording flag +0x35, posts ArpSeqMode = 0 (UI id 0x19b), then makes the page the overlay object for 0x47868c00 ms.
- RecordPage__stop (0x42022908): logs "Stop Record", clears the flag, removes the overlay (UI__setOverlay with 0 and 400), posts ArpSeqMode = 2 (Seq). RecordPage__stopIfRecording (0x42022950) calls it only while recording.
- RecordPage__onEvent (0x42022960) item lists dumped: Off {0x19b}; Arp {0x19b, 0x1a0, 0x1a2, 0x19c, 0x19d, 0x1a3, 0x1a1, 0x19e, 0x19f}; Seq {0x19b, 0x1a0, 0x1a2, Record, Edit, 0x1a4, 0x1a3, 0x1a1, 0x1a5, 0x1a6, 0x1a8, 0x1a7, 0x19f}.
- Corrections to the earlier text: kind 7 tests the control id (0x19b); byte +40 is a browse byte (1 = list navigation, 0 = value editing), so the encoder edits the selected item by default and moves the selection only after an encoder press.
- Slot delete callbacks 0x420215d8 and 0x42021620 stay medium: the pairing with a modal on the sound list is inferred from the shape.
- Still unconfirmed: which routine consumes incoming notes while the record page is the overlay object, and the caller of PlayPage__stepOctave.

## Round 67 - Front panel handlers

- Dispatcher fcn.4201e640: kind 0 calls FrontPanel__onButton (0x4201e45c), kind 2 calls the pot section chooser fcn.4201e390, kind 3 calls FrontPanel__encoderSections (0x4201e4f4); kinds 12 and 13 reset 15 pot cache entries at +0x270 (stride 20, s16 = -1) and stamp +0x26c. Then the section handlers (oscPitch, chord, wave, ...) are called with the active flag.
- Front panel fields: +0x260 section shown (0 = none), +0x264 selected section in the ring (starts at Wave +0x70), +0x268 debounce stamp, +0x26c activity stamp. Earlier names "lock word" and "pending state word" for +0x260 were wrong. The 1000 ms test is an inactivity test (switch to page state 2), not a hold timer.
- Wave section (0x4201f6a0): pot 2 = wavetable position (UI 5/12) or with shift osc volume (UI 3/10); pot 7 = wave envelope amount (UI 23/25) or with shift velocity amount (UI 24/26); encoder 2 = wavetable (UI 4/11); button 10 or 12 enters a details menu with the LimitWT step (UI 6/13). Osc selection byte ui+0x1294: 0 Osc 1, 1 Osc 2, 2 both (inferred from the indices written).
- Osc/pitch section (0x4202049c): pot 0 = detune (UI 1/8), pot 1 = glide rate (UI 16), encoder 1 = semitone (UI 0/7), details menu (button 10 or 11) with PitchBendRange (UI 18), Osc keytrack steps (UI 2, 9). Encoder 0 changes the osc selection; when the displayed index is 16 it also writes GlideMode (UI 17).
- PanSection__draw (0x4202019c) stays medium: the surface-derived arguments (+0x401, +0x404) of the text draw fcn.42028f30 are not resolved.

Addendum to Round 67 (the four unconfirmed items re-checked):
- Osc selection byte ui+0x1294: 0 = Osc 1, 1 = Osc 2, 2 = Link (both). Confirmed by the Osc and Wave draw routines (labels Osc1, Osc2, Link at x = 0, 46, 92; Wave details titles Osc 1 - Details and Osc 2 - Details). The byte is only used by the Wave and Osc/Pitch sections (plus its zero init in the UI constructor).
- Osc/Pitch encoder 0 with the glide rate shown: the page draws a Mode: row (GlideMode) beside the rate, so the GlideMode write is intended. The osc selection byte is changed first in every case, so the same turn also moves the hidden osc selection. That side effect is as the code reads; whether it is intended is unknown.
- Text draw arguments: surface+0x401 is an unused empty helper object address; surface+0x404 = ui+0x16ac is an embedded font object (first word 0x3c073b40, font 1; ui+0x16b0 holds font 2, 0x3c07423c), both set up by the UI constructor. PanSection__draw is now strong.
- Kinds 12 and 13: in the UI dispatcher they set ui+0x28 = 1 (pot and encoder events clear it); in the front panel they reset the 15 pot cache entries and stamp +0x26c. No code sends to the UI queue 0x3fccff64 by direct reference, so their producer and meaning stay unconfirmed. The same applies to the receiver of queue 0x3fccff60 (the target of UI parameter writes); neither queue handle is referenced by literal anywhere else.

## Round 68 - Tempo clock, rhythm clock and scale mask

- ParamInfo descriptor table (0x3c071f54, 32 bytes per entry): +0 name, +4 label, +8 steps, +12 enum label table, +16 default, +20 second float, +24 flags, +28 formatter function. The formatter of an entry sits at entry+28, i.e. just before the next entry's name. Formatters read here: 0x42017784 SeqStep1Pitch ("%+d" of n - 48), 0x420177b0 RhythmPattern and SeqLen ("%d" of n + 1), 0x42017850 ArpOctaves ("%d Oct" of n + 1), 0x42016ae0 BPM (not read). All round the float with +0.05 in double and truncate.
- Enum labels: SeqRoot (13) = -, C, C#, D, Eb, E, F, F#, G, Ab, A, Bb, B; SeqScale (24) = Chromatic, Whole Tone, Diminished, Maj Penta, Min Penta, Major, Minor, Dorian, Phrygian, Lydian, Mixolydian, Locrian, Melo Min Asc, Harmonic Min, Lydian Aug, Lydian Dom, Mixolyd 6b, Half-Dim, Altered, Blues Minor, Blues Major, Octatonic, Min 3 + Min 2, 4th + Min 2. Defaults: SeqLen 16 (stored 15), ArpOctaves 3 (stored 2), BPM 120.
- TempoClock__update re-read: the byte at +32 is a sync-BPM-in-use flag (logs on each switch), not a tempo-changed flag. TempoClock__initState (0x42016b28) also sets the double 120.0 at +8/+12.
- RhythmClock__swingStepLength (0x42017ff8) confirmed: step length = base x 2s on even steps and base x 2(1 - s) on odd steps, s = 0.05 + 0.9 x clamped to 0.05 .. 0.95, parity = bit 0 of the step counter at +88. The earlier doubt is resolved.
- RhythmClock__resync is called only by RhythmClock__loadStepLength, when the cached lock byte +49 differs from the current lock state.
- Arp__updateScaleMask (0x42018478): rebuilds the scale mask only when the scale byte (+0x28e) changes; a root change alone (+0x28f) does not trigger it. Root 0 means no root and sets the flag +0x291. It is called from the sequencer step function and from Sequencer__start (0x420184e8, slot +16 of vtable 0x3c0731a8). Unconfirmed: the offsets +0x28e and +0x28f are taken to be SeqScale and SeqRoot from the engine id order (0x282 + id); the parameter store was not traced.

## Round 69 - NoteProcessor forwarders and engine layer broadcasts

- The static initialiser 0x42014694 builds two globals: the Engine at 0x3fca7508 (vtable 0x3c070218, size 0x3a70, four voice managers at engine+8 + i x 0xcb8, layer active byte after each manager, destructor registered) and the NoteProcessor at 0x3fca5828 (vtable 0x3c070110, secondary vtable 0x3c07018c at +4, engine pointer at +8, TempoClock at +0x10, transport record copy at +0x70, arp object at +0x728, own scale mask instance of 12 records at +0x1bf8). The earlier text that merged both as "layer voice managers of the object at 0x3fca7508" was incomplete.
- NoteProcessor note forwarders (primary vtable slots +20 and +24, secondary base at primary+4): velocity float x 127 truncated, channel sign-extended from 7 bits, layer -1, then the engine noteOn (fcn.42014cec) or noteOff (fcn.42014fcc) with the original float on the stack. The noteOn argument a4 is not forwarded.
- Engine vtable broadcast slots: +44 setSustain (flag, layer or -1), +52 active voice count, +56 setPolyphony (limit 1..8, layer or -1), +68 / +72 / +76 setters of layer+0x3c0 / +0x3c4 / +0x3c8 (engine events 7, 8, 9; meaning unconfirmed), +80 allNotesOff, +84 fcn.4204e7b8.
- Voice manager fields confirmed: +0x392 polyphony limit (1..8), +0x398 sustain pedal flag, +0x62c 128-byte note state table (2 = held by pedal), state byte of slot k at +0x11 + 0x70 k, +0x3c0 word, +0x3c4 word, +0x3c8 byte. Sustain release: only when the old flag is 1 and the new value is 0 does it send noteOff for every note in state 2.
- The scale mask object (fcn.42017348) is 12 records of 8 bytes plus root mod 12 at +96 and set-flag count at +100; the sequencer has one at +0x29c, the note processor one at +0x1bf8.

## Round 70 - Parameter info accessors and engine static init

- Correction to Round 69: the vtable that holds the note forwarders is the NoteProcessor vtable 0x3c070110 itself. The address 0x3c070168 is slot +88 (VoiceLayer__copySnapshot), not a separate vtable. fcn.42013d50 and fcn.42013da0 are the float-velocity overloads of noteOn and noteOff at slots +108 and +112 (the integer overloads are slots +20 fcn.42014528 and +24 fcn.42013e90). fcn.42013d78 and fcn.42013dc8 are slots +12 and +16 of the secondary vtable 0x3c07018c. The float overloads call the engine noteOn and noteOff directly, bypassing the chord and arp logic of the integer noteOn.
- UI parameter index to descriptor (shared by the six ParamInfo accessors): below 0x150 the 28-byte entry (index mod 84) of the table at 0x3fc9fb78; 0x150 to 0x230 the entry (index - 0x150) of the table at 0x3fc9e2dc. Entry: +0 four-character tag (for example o1s_), +6 u16 engine parameter id (0xffff = none), +8 u16 effect type index, +12 fallback label or name, +16 fallback default, +20 fallback step count byte. The record comes from the engine descriptor getter (fcn.42014f7c, via NoteProcessor slot +12) when the engine id is at most 0x1e0, else from the 32-byte effect parameter table at 0x3c06f5c4 (type index at most 74). Record fields: +0 name, +4 label, +8 steps, +12 enum labels, +16 default, +24 bipolar byte, +28 formatter.
- Engine descriptor getter fcn.42014f7c: ids 0..20 give entry id of the table at 0x3c071f54; ids 21..148 (128 sequencer step parameters) share entries 21..24; ids above 148 are sound parameters: layer = (id - 149) / 83, parameter = (id - 149) mod 83, descriptor from the layer voice manager.
- Engine static init (0x4200d6a0): audio engine object 0x3fca3974 (first two words load average and peak); binary semaphore at 0x3fca3964; event queue of 0x400 x 24 bytes at 0x3fca3960; the engine log line "Constructor called" and the cached NoteProcessor pointer in 0x3fca5010 come from fcn.4200d754. The earlier text calling that pointer the engine pointer was wrong.
- Unconfirmed: meaning of the float block at engine+0x1600 and of the global 0x3fca37b0 (holds a double 120.0 at +0x188).

## Round 71 (held-key helpers, small statics and stubs, sysex sender, serial console)

- Held-key helpers verified by reading the code: HeldKeys__prevByPressTime (0x4204ead0) is the mirror of nextByPressTime (pairs of 64-bit press time and note number, no wrap-around, -1 if none); HeldKeys__lowestVelocity (0x4204ebd8) keeps the lower note on equal velocity, highestVelocity keeps the higher. Clock__stopStub (0x4204ed20) is empty and only called from ArpRunner__stop.
- ParamInfo formatter 0x4200f9b4 belongs to the descriptor entry CompressorRelease (entry at 0x3c06fb64, formatter field at +28): "%.0f ms" of value x 2 x 1000. Renamed ParamFormat__compressorReleaseMs.
- Voice class: slot +8 (0x42015968) returns 0x3c072274 + 32 x index (index 0..82); 0x3c072274 is entry 25 of the ParamInfo table 0x3c071f54. Voice object size is 0x828 (deleting destructor 0x42016088, pool address 0x3fcaaf78 + i x 0x828 in 0x420162bc). The timbre class deleting destructor 0x42016638 frees 0x4074 bytes. The console prints these as sizeof(SynthVoice) = 0x828 and sizeof(SynthTimbre) = 0x4074, and sizeof(dsp::MW1WavetableCache) = 0x2008 (names taken from the firmware strings).
- Static init entries 0x42029248 and 0x42029f28 each create a mutex with fcn.40384c7c(1) (84-byte control block, length 1, given once). Handles at 0x3fcd32b0 and 0x3fcd45c8.
- 0x4204e628 is NoteProcessor slot +88 and copies the transport record into this+0x70. Renamed NoteProcessor__setTransportRecord (the earlier name VoiceLayer__copySnapshot was wrong). 0x42011be0 is the EQ reset (slot 1 of the EQ vtable 0x3c070050). 0x4204dd14, 0x4204de34, 0x4204e47c are empty slot 0 stubs of Drive, EQ and Phaser, called from Part__buildEffects.
- Sysex sender 0x420282b0: sends a length-prefixed block on port 0 as 4-byte packets (first byte 4 with three data bytes, final packet 5, 6 or 7), retries a failed push after one tick delay up to 2000 times. Port table 0x3fcd6660 has 480 bytes per port. MidiPort__isReady = 0x4202d058 (byte +1 non-zero and byte +2 non-zero), MidiPort__pushPacket = 0x4202d0fc (needs 4 free bytes in the FIFO at port+36, then 0x4202d9b0 writes and 0x4202cfd0 starts the transfer). Caller is the MIDI/control task fcn.420191ec. That the port is USB follows from the packet layout only.
- Serial console Console__periodicReport (0x42025074): fully read. It is a debug command interpreter on stdin (Console__readChar 0x420407a0). Commands A, l/L, y/Y, E/e, m/M, n/N, c/C, S, X, i/I, #, p, comma, period, and the note chord keys are listed in the function map row. Notable findings: the MIDI messages B0 7B 00 and B0 78 00 are injected through the std::function at UI object +0x17f0; note-on/off events {1 or 0, note, 108} through the std::function at UI object +0x17e0; the 16-note series is 60 64 67 71 72 76 79 83 84 88 91 95 96 100 103 107.
- Not confirmed: what the flags behind E/e (16 words at 0x3fcd1ce4), m/M (UI +0x17cc) and c/C (0x3fcd2d78+184) do; what UI event kind 35 does in the dispatcher; the tables behind the p command (0x3fcd0910, 0x3fccf428, index 17); the meaning of the third event field 108; the per-slot values printed as "#%d = %.2f" (table 0x3fcce3d4, 8 entries of 12 bytes).
- Still deferred by the user: producers of queues 0x3fccff64 / 0x3fccff60 and the meaning of dispatcher kinds 12 and 13.
- Correction to Round 48: 0x4204e918 and 0x4204e93c do not include idx itself. They return the next held note strictly above (HeldKeys__nextHeldAbove) and strictly below (HeldKeys__nextHeldBelow) idx, and -1 for idx above 127. Both are now strong.

## Round 72 (last medium rows)

- Sound list: the Actions menu handler at 0x420218ac is now named SoundListPage__onActionChoice. It builds the modals and shows that the two delete callbacks are the confirm callbacks of "Delete Preset" (case 1, 0x420215d8) and "Delete Presets" (case 2, a range of two slot numbers 1 to 360, 0x42021620). Case 0 is the Category Filter modal (callback 0x4204ef48 stores value-1 in byte +96; category names at 0x3fca0560, 18 entries), case 4 Send Presets (0x42021584 posts type 20), case 5 Receive Presets with "Store at:" (0x420215b0 posts type 24); cases 3, 6 and 10 post the events {0,0,19,0}, {0,0,24,0} and {0,0,26,0} without a modal. The case-4 button text was not read, so the description row may say more than the code shows about it. Not confirmed: the menu label for each choice number, the effect of UI event types 19, 24 and 26, and fcn.420294bc.
- FrontPanel__onEvent (0x4201e640) read in full: step 1 by event kind, then a fixed fan-out to 12 section handlers with a per-section flag (input flag and section shown). Details in the map row. Kinds 12 and 13 stay deferred.
- NoteSlots__init (0x4204ed50) is called from the arp setup fcn.42017718 on the member at arp+0x13d0 and is now strong; no medium or low Protein rows remain.

## Round 73 (rating recheck, group 1: storage and persistence)

- 17 rows moved from "notes" to read-from-code: Storage__init/load/save/erase, loadSound/saveSound/soundSize, loadGlobalsPartial/saveGlobals, Global__load/autosave/set, Settings__defaultEntry, Sound__requestSave, Sound__saveWorker, LibraryIndex__build/saveSlot.
- Correction to the older notes: NVS partition is "data", sound blobs use namespace "sounds" with keys "sound-%03i" (blob = 32-byte header + 561 records of {tag, float} = 0x11a8 bytes), and the settings blob uses namespace "globals", key "default" (134 words = 0x218 bytes). The notes said the namespace is "data".
- Observed in the code and not confirmed as intended: on NVS error 0x110d or 0x1110 the init erases and re-initialises the default partition "nvs", not "data". Storage__init opens "sounds" read-only and leaves that handle open.
- Settings records: the 24-byte table at 0x3c0752a0 (134 records) holds minimum at +8, maximum at +12 and default at +16.
- Recheck status after this group: 248 rows still rated "notes", 516 verified.

## Round 74 (rating recheck, group 2: boot and hardware init)

- 15 rows read from code: ELF entry (Startup__entry), App__init, Engine__init, Board__init and its helpers, Display__init, MidiUart__init, Usb__init, Audio__initI2s, Audio__writeBlock, Board__setLedBrightness, Gpio21__set. 0x42032d94 is the IDF function esp_efuse_mac_get_default and was moved from Protein to library.
- Pin map read from the code: shift register clock GPIO 11, latch or load GPIO 12, data out (LEDs) GPIO 13, data in (buttons and encoders) GPIO 14; LED brightness PWM on GPIO 1 (LEDC timer 0, 8 bit, 20 kHz, output inverted); display reset GPIO 45 and the SPI display with an SSD130x-style command set (column-wise addressing); I2S MCLK 7, BCLK 9, WS 10, data out 8, 32-bit stereo, 2 DMA descriptors of 64 frames; DIN MIDI UART 1 at 31250 baud, TX 17, RX 18; GPIO 6 and 21 are plain outputs at level 0. The I2S pin order is now read from the IDF config struct and no longer assumed.
- Engine__init creates the "Extra Engine task" on core 1 (entry fcn.4200ccc8, priority 24, stack 0x4000) and notifies it once.
- Not confirmed: what GPIO 6 and GPIO 21 are wired to, the exact display controller chip, the unidentified ROM cache setup calls in the startup entry, and the two library calls in Board__startAdcTask.
- Not read yet: the USB MIDI input reader fcn.420280b0 (292 bytes), left for the MIDI group.
- Recheck status after this group: 232 rows still rated "notes", 531 verified.

## Round 75 (rating recheck, group 3: MIDI and sysex, small and mid-size functions)

- 18 rows read from code: CC map assign and clear, sysex thunk, the three UI queue functions (processEvent, processMidiInput, createQueues), the queue pump, DIN and USB short and block senders, MIDI output task, library used-flag getter, Global__reset, the MIDI system message handler, the transport record builder, the CC handler, the message router and the USB MIDI input poll.
- Queues (all 128 entries): 0x3fccff64 (16 B) is filled from controller code through UserInterface__processEvent (log "uiEventRxQueue full!"); 0x3fccff60 (16 B) is written by the UI event post 0x4201d010 and drained by the pump 0x4201c6c0 into the ring buffer object 0x3fcd3298; 0x3fccff5c (11 B) takes incoming MIDI (log "uiMidiMessageRxQueue full!"); 0x3fccff58 (11 B) holds outgoing short MIDI that the pump sends to DIN and USB; 0x3fccff54 (24 B) holds engine events that the pump passes to Engine__sendEngineEvent. An 11-byte MIDI item is 3 MIDI bytes plus an 8-byte timestamp.
- Observed and not confirmed as intended: the DIN length table at 0x3c073a48 gives 3 bytes for channel pressure (0xD); the MIDI standard has 2. Correction to an earlier reading of that table (status 0xC is 2, 0xD and 0xE are 3).
- MIDI clock: 24 ticks per quarter note, timestamps behave as microseconds (tempo = steps * 2500000 / delta), tempo accepted from 5.0 to 1000.0 BPM, estimate refreshed after 500000 units. Clock messages need the enable byte at +0x220.
- Router: events are {type, a, b, c, d, e} words sent to Engine__sendEngineEvent; CC events are {3, cc, value, part, channel, 0}. Program change only stores the accepted program number as s16 at controller+0x1af4 (consumer not read). CC map conversion rules are in the CC handler row.
- Not confirmed: meaning of the router extra argument (a6), roles of global settings 0 and 1 (omni and base channel by use), the MPE state machine 0x42028f70, who fills the output queues 0x3fcce43c and 0x3fcce438, who reads the USB input item ring and the sysex buffer, the byte at clock object +0, parameter offsets 5 and 12 of each 84-parameter block.
- Still deferred by the user: dispatcher kinds 12 and 13 and the consumer of ring 0x3fcd3298 (partly answered above, not chased).
- Recheck status after this group: 214 rows still rated "notes", 549 verified.

## Round 76 (rating recheck, group 4: MIDI core)

- 7 rows read from code: Controller__handleEvent (0x4201a0d4), Controller__init (0x420193ec), Controller__initState (0x4201a90c), MidiSysexController__processMessage (0x4201bbf8), Mpe__process (0x42028f70), the two word-table accessors (0x4204f8d0, 0x4204f8f0); the router row (0x42019bdc) was rewritten with the new findings.
- Event flow (answers the deferred queue question, read from code): UI event post 0x4201d010 -> queue 0x3fccff60 -> pump 0x4201c6c0 -> ring buffer 0x3fcd3298 -> the output task 0x4200ca8c pops one 16-byte item {id, value, kind, flag} per loop and calls Controller__handleEvent -> the handler forwards the original four values to the UI through UserInterface__processEvent -> queue 0x3fccff64 -> UI dispatcher 0x4201d680. The handler also puts its own follow-up items in the ring: kind 12 (sound loaded, value = slot or -1), kind 13 (layer-copy load done, value = slot) and kind 18 (slot deleted, value = slot). These reach the UI dispatcher through the forward. Controller kinds handled: 0, 6, 8, 9, 10, 15, 17, 19, 20, 22, 24, 25, 26, 32, 34, 35 (details in the map row).
- Correction to the older notes: sysex command 0x14 is the preset DELETE request (string MSGID_PRESET_DELETE_REQUEST), not select or check; it ends in controller kind 17.
- Router fifth argument: 2 marks the startup demo player (Midi__injectNoteOff passes 0, the demo player 2); with 2 the channel check is skipped.
- Byte this+0 of the controller (used by the transport record builder) is set from global setting 10 (nonzero = 1) by controller kind 32; the clock-enable byte +0x220 is 1 after init.
- MPE: zone member count of the lower zone is global setting 132 (copied to MPE+64 on every message), the any-zone byte MPE+74 is set to 1 in Controller__init, RPN 6 on the master channels changes it. Mpe__process writes its result word only when the message is accepted; the router reads that word in every case (observed, impact not confirmed). Upper-zone messages are dropped by the router.
- First boot: the flag "Factory"/"Data" missing puts ring item kind 26 {0,0,26,0}, saves the flag, and shows "Protein / Init Factory / Presets ..." on the display. Controller kind 26 (also from the Actions menu) restores factory presets via fcn.42029f08 with the text "Init Presets ...".
- Boot posts first events: kind 9 with the last sound slot (global setting 9) or kind 8 when it is negative, then kind 32.
- Not confirmed: who consumes the s16 words at controller+0x1af2 and +0x1af4, the factory restore internals (fcn.42029f08), whether kind 5 items are a text popup, the 2-bit fields of the kind 10 id, the caller of Controller__initState (probably a static initialiser).
- Recheck status after this group: 207 rows still rated "notes", 556 verified.

## Round 77 (rating recheck, group 5: effects)

Rows rechecked against the code: FX base, parameter setter, FX chain, compressor, chorus, flanger, phaser, tremolo, drive, EQ, delay, reverb (45 rows). Result: 35 rows verified (strong), 9 rows rated medium (structure read, arithmetic only partly interpreted), 0x4203d85c reclassified to library (libm powf wrapper). Details are in the map rows; the findings that change earlier notes:
- FX chain (Fx__process): bus 0 -> FX1 -> add bus 1 -> FX2 (skipped when the FX2 pointer is null or equals the FX1 pointer) -> add bus 2 -> master biquad (pass-through, b0 = 1; no writer of the coefficients was found) -> peak limiter (threshold 0.85, attack 0.2, release 0.002, smoothing 0.01) -> Fx__clipBlock (+/-infinity to 0, hard clip to +/-1, NaN passes).
- FX parameter setter: complete id -> field table for ids 0..74 is in the Fx__setParam row; type table order is Bypass, Chorus, Flanger, Phaser, Tremolo, Drive, Compressor, EQ, Delay, Reverb.
- Compressor kernel: the peak detector has no absolute value (negative half-waves are not detected); detector 1 coefficients 0.6 / 0.002 are fixed; Attack / Release (ids 44 / 45) are the second-stage coefficients powf(0.1, 1 / (0.2 or 2 * x * 11025)); gain = ((threshold + env) / threshold)^(1/ratio - 1); AutoGain make-up = 0.8 * dB(thresholdDb / ratio - thresholdDb); look-ahead delay line of 92 samples per channel with write index +0x370 = read index +0x374 + n (n multiple of 4, max 84).
- Chorus: level compensation after the sample loop scales ALL 16 samples of both channels (the older note that only indices 0, 4, 8, 12 are scaled was wrong); sign and constants of the compensation: g = 1 - 0.13 * stageIndex, minus 0.25 * fb when fb > 0, plus 0.2 * fb when fb < 0. The sine LFO reads the nearest table entry without interpolation (older note said linear interpolation). Depth = d^3 * 882 samples, Speed rate = 0.001 + 20 * s^2, Center delay = 2 + 8820 * c^2 + depth; voices per channel = Stages index + 1.
- Filter coefficient discrepancy to check: chorus, delay and tremolo filters compute expf(-pi * 0.5 * x^3) according to the register trace (the 0.5 is the value held in a register in every path); the older notes say expf(-pi * x^3). Marked unconfirmed in the rows.
- Flanger: depth = d^4 * 1411.2 samples, rate = 0.001 + 20 * s^4, base delay = 1 + 1102.5 * c^2, delay constant within a block, buffers 2,613 floats, feedback gain curve 1 - (1 - v)^3 (v = 2 fb - 1), compensation 1 - wet * 0.45 * |gain|.
- Phaser: sections per channel from table [1, 2, 4, 6, 8] by Stages index; two 256-entry tables are built in the constructor (A: 11.56 * 1626.5^(i/256) / 44100, B: 2 * sin(pi i / 256)); Allpass2__setCoeffs (0x4202a2b0) builds the 2nd-order all-pass; per-model section formulas are only partly interpreted (medium).
- Tremolo: gain = ((1 - d) + d * (lfo + 1) / 2)^2, free rate = 0.1 + 39.9 * s^2 Hz, shapes sine (interpolated) / triangle / square with one-pole smoothing; the tempo-sync branch is only reachable when the word +0x74 >= 0.1 (0 after the constructor, writer not found).
- Drive: type index 0..4 shaper formulas are in Drive__shape; Amount below 0.125 (after squaring) crossfades to a clean pass-through; wet = 2m - m^2, dry = 1 - m^2; output gain = -20..+20 dB from the Gain parameter.
- EQ: four Zoelzer-form biquads (low shelf 30..450 Hz, two peaks 200..7000 Hz with Q 0.1..16, high shelf 1.65..20 kHz); every band gain and the output gain equal (parameter - 0.5) * 40 dB * m where m = DryWet; there is no dry / wet audio mix.
- Delay: buffers 441,000 floats; free time = 1 + 88200 * x^2 samples; sync ratio table 0x3c06ffd0; slew (target - current) * 0.000125 per sample; DC blocker 0.99, 1, -1 plus High Cut / Low Cut in the loop. Stereo routing and the output law are only partly interpreted (medium).
- Reverb: ReverbCore__setPreDelay maps PreDelay x to trunc(16370 x (1 + x)) limited to 32736 in steps of 16 samples; shared ring of 65,536 floats; seven early-reflection taps per channel; the internal network (diffuse, process, Time / Color mapping) is only partly interpreted (medium).
- 0x42018aec is ParamDesc__globalIndex: 28-byte descriptors, returns the global setting number (or -1) for a parameter id.
- A tiny function at 0x4202b59c (no map row) sets the table lengths of the sine / dB table object to 0x400.
- Not confirmed: writers of the optional modulation-source pointers in every effect (all cleared in the constructors), the tempo-sync inputs of the tremolo and delay, the unit of the delay beat length, the model-specific formulas of the phaser, the reverb internals, the unused words +0xf4 / band words of the EQ.
- Recheck status after this group: 162 rows still rated notes, 591 verified (strong), 9 medium.

## Round 78: rating recheck, group 6 (arpeggiator and sequencer, 0x42017348 to 0x42018858)

All 14 rows of the group plus the two attach slots (0x4204ecf8, 0x4204ed40) were read in full from the disassembly and rewritten (map150, map151). Corrections and additions to the round 3-4 notes above:
- Scale snapping (Sequencer__step): the note value is snapped UPWARD to the next scale pitch class (not "nearest"), and the snap happens before the -48 offset. With a root set the absolute pitch (base + value) is snapped; with root "-" (no root, flag +0x291) only the value is snapped and the base note is added afterwards. The mask object (ScaleMask__build 0x42017348) is rotated by the root: record i is set when mask[(i - root) mod 12] of the 24 x 12 byte table at 0x3c071e0c is set; names come from 12 spelling rows at 0x3c071bcc selected through the offset table at 0x3c071b6c.
- Arp algorithms (ArpGen__step): 0 Up, 2 Up-Down and 3 Up|Down start low and go up; 1 Down, 4 Down-Up and 5 Down|Up start high and go down (not "0/3 up, 1/4/5 down"). 2 and 4 turn without repeating the end note, 3 and 5 repeat it. The octave counter walks 0 to count-1 on the way up and back on the way down. Algorithm 6 Random: held note from ArpGen__randomHeldNote (lowest held note plus rand mod 20 upward steps over the held notes, wrapping) and a random octave.
- Note order type 6 (chord) is the lowest-velocity held note followed by up to three higher held notes, all plus the octave offset; it is unreachable while the order float stays 2.0. Order type picks the sweep start through HeldKeys__firstNote_forward (0x4201787c) and HeldKeys__firstNote (0x4201799c).
- ArpReset: the generator limit is ArpReset + 1 steps (0x7fffffff when ArpReset is 0); at the limit the runner calls the generator reset slot and clears the clock step index. The sequencer never resets this way. The unit and labels of the ArpReset choices were not checked.
- Runner (ArpRunner__step): steps whose rhythm pattern byte is 0 or 2 return before the generator is called, so the arp does not advance on rest steps. Velocity = generator velocity x the clock float (always 1.0, the per-step floats are never changed). The tie byte is output by the generator for the step just played and makes the NEXT step a tie continuation: its generator call still advances the position but its notes are dropped. This cannot happen in normal use (no step type or pattern value 2 is ever written).
- The sink calls are the float overloads at slots +12 and +16 of the NoteProcessor secondary vtable (noteOnFloat_secondary, noteOffFloat_secondary), so arp and sequencer notes bypass the chord and arp logic of NoteProcessor__noteOn. The channel argument is the signed byte at held-key object +0x680 (obj+0x720); note-on also clears byte A+0x1436 and sets A+0x1437 (via a 16-bit store of 0x100). The sounding-notes table at A+0x1438 holds 100 after a note-on and 2 after a note-off.
- ArpSeq__tick: the flag byte +0x14b8 toggles while no key is held (set and table cleared on one tick, cleared without a scan on the next), so the sounding table is cleared on every second tick then. A mode change while enabled stops the runner, sends all-notes-off and re-inits the slot table. New-key byte +0x681 is only read here; who clears it was not checked.
- Sequencer__reset (SeqReset): 0 Start = position 0 (last step for backward), 1 None keeps the position (a finished forward-once position becomes 0), 2 Random = rand mod length; also clears the published step floats and sets the last-base-note byte to 0xFF. A new newest key restarts the position only for SeqDir 4; otherwise the sequence continues.
- Formatters: 0x42017758 prints value x 100 with "%.0f%%"; 0x420177dc prints "<step+1> <name>" and confirms the older wavetable finding (steps 32 to 63 show tables 56 to 87, User tables skipped).
- Still unconfirmed: writers of the generator floats at +8 (order, 2.0) and +20 (velocity source, 0); consumers of the step floats at seq+0x308 and +0x30c; meaning of byte A+0x1437; clearing of the new-key byte +0x681; the label set of ArpReset.
- Latch note (design only): the runner stops itself when all 32 held-flag words are zero (ArpSeq__checkHeldStart) and the tick clears the sounding table while no key is held, so a latch must keep the held flags, velocities and press times set (as already stated in Round 52).

Recheck status after this group: 146 rows still rated notes, 607 verified (strong), 9 medium.

## Round 79: rating recheck, group 7 (parameter and preset functions, 0x42018a50 to 0x42019024 and 0x4202925c to 0x42029f5c)

All 13 rows were read in full and rewritten (map152, map153). Findings and corrections:
- Missing versus empty presets: a slot with no stored blob (size 0) or an unreadable blob makes Sound__load and Sound__loadLayer return 0 and leave the sound unchanged. The "Init" name, category 0 and version 8 (bytes +0x8dc/+0x8dd = 0/8) with all 561 parameters at their defaults happen when one of the first 4 header bytes is 0xFF (empty header). Round 17 attributed this to a missing blob.
- Sound__load (full load) first writes defaults for all 561 parameters and the name "Init", then overlays the stored records; a parameter whose tag is not in the blob keeps its default. Sound__loadLayer does not write defaults: it only copies the records of the source layer into the destination layer (per-layer parameters only, global ones untouched) and then still replaces the sound name, category and version byte from the slot's library header, so a layer load renames the sound.
- Both loaders use ParamInfo__findByTag (linear scan of 561 entries), Sound__migrateValue with the version byte at blob+25, and the library index header (object 0x3fcd32b4, 32 bytes per slot) for name and category. Callers: full load 0x420198c2, 0x4201a2d9, 0x4201b1ea; layer load 0x4201a3e2.
- The engine record default is at record+20 (ParamInfo__defaultValue reads it); Round 69's "+16 default" does not match the code (record +16 is not read by any of these functions). Fallback defaults come from descriptor+16. UI index 83 (layer A, Active in Multi) is forced to 1.0; the same slot in layers B to D takes the record default of engine id 231 (value not read).
- Engine ids: UI index below 0x150 maps to 83 x layer + the descriptor id (layer A starts at 149); the global table gives its s16 at +6 directly. ParamInfo__tag/findByTag use three name characters plus the layer digit for per-layer parameters and four characters for global ones.
- ParamInfo__formatValue: a record formatter (record+28) wins; otherwise stepped parameters print the label (record+12 or descriptor+24) or the index, bipolar ones "%+.2f" of (v - 0.5) x 2, others "%.2f".
- Globals__formatValue: LED brightness (setting 7) is printed raw, not +1 as the round 3 item list says (Display brightness 6, Receive Chan 1, Send Chan 2 and MPE Max Channel 132 are +1). Not checked whether another path shows LED brightness.
- Sound__migrateValue (0x420294f0): operations A to G and the version table of Round 36 were re-checked against the code (constants, thresholds, tables at 0x3c075f30 and 0x3fccde98, enum remaps for FilterType in versions 6 and 7); the full per-version branch network was not retraced, so the row is rated medium.
- Factory presets: Presets__loadFactory maps the "approm" partition (type 0x40, subtype 0) with AppData__get and feeds it through Sysex__feedStream into slots 0, 1, 2 ... Sysex__feedStream keeps a 1 KB message buffer, waits 10 ticks after each message and saves only when the sysex handler sets the byte at 0x3fcce69a (meaning not established). Cache words: mapped pointer 0x3fcd45d4, partition 0x3fcd45d0, mmap handle 0x3fcd45cc (Round 14 note listed the last two the other way round).
- Still unconfirmed: what sets 0x3fcce69a; the record word at +16 of the engine descriptor; the default of the Active slot in layers B to D.

Recheck status after this group: 133 rows still rated notes, 619 verified (strong), 10 medium.

## Round 80: rating recheck, group 8 (front panel and UI, 0x4201bab4 to 0x42026198)

All 64 UI and front-panel rows in scope were read in full and rewritten (map154 to map157; 67 rows in total with the two display primitives Framebuffer__fillRect and Text__drawRightAligned and the sysex version sender). Findings and corrections versus the older notes:
- Event path: every UI edit goes through UI__sendEvent as {param index, value x 1e6, type 6, 0}; a UI index below 0x150 is part x 84 + the layer parameter, 0x150 and above is global (0x19b ArpSeqMode, 0x1a4 SeqLen, 0x1a9 + 4n step pitch, 0x1aa + 4n step velocity, 0x229 LayerMode, 0x22a LayerAlloc).
- Page objects and draw/input conventions (ui+0x2c sound list, +0xb8 save, +0x198 front panel, +0x534 play, +0x548 arp/seq, +0x584 step editor, +0x5ac layer, +0x75c header, +0x780 settings, +0x790 info, +0x79c mappings, +0x7ac debug, modal +0x878); Page__idleTimeout (vtable offset 36) returns to the front panel after more than 999 ms of no input.
- Front panel: 15 pot cache entries (20 B each), pot dead-band of 30 raw counts, anchor reset after the first accepted change; pot ids 13 and 14 do nothing in the section chooser. The envelope group view shows percent (value x 100 + 0.05 as "%d"), not time; the Pitch section's grid label is "Pitch".
- Highlight rule used by all pages: steady while the browse or navigate flag is set unless both shift bytes are set, blinking otherwise (BlinkPhase__tick, toggles every 8 counts).
- Save page: 360 slots (0..359), 20 categories (0..19), name up to 24 characters through the text entry (77-entry character table), store sends event type 15 (save to slot).
- Layer page: Copy and Paste cover indices 0..82 of the part (index 83 ActiveInMulti is not pasted); Initialize writes ParamInfo defaults for the same range.
- Recorder: first recorded note is the base, pitch = note - base + 48 folded to 0..96, 32 steps maximum.
- Info page prints "Version 1.03 b3" from code immediates while the image is labelled v1.02 (unexplained).
- Still unconfirmed: meaning of event types 12 and 13; initial s16 values of the Chord, Pan, LFO and FX sections; why wavetable tables 32 to 55 are skipped in the step formatter (inferred user slots); entry byte +24 of the sound library entries (category, inferred); the reset of the MasterSection counter at +38; the cursor semantics of the text entry (inferred); that SeqLen holds length minus 1 (inferred); content of the third debug buffer (+140); the purpose of the last Sound__setName argument (8) in SavePage__store; callers of RecordPage__onNote; the single-layer LED mapping in LayerPage__updateLeds (inferred).
- Not part of this group (separate later group): board, test mode, display driver and ADC rows (0x42026238 to 0x42028f30, 0x4204f1e0, 0x4204f3ac) and the text blit helpers 0x42028e0c and 0x42028c40.

Recheck status after this group: 66 rows still rated notes, 685 verified (strong), 11 medium.

## Round 81 - Recheck group 9: engine and voice render (43 rows)

Scope: engine task, audio output task, engine event dispatcher, note processor forwarders, voice manager, voice DSP object, LFO update, constructors, tempo clock. All 43 rows are now described and no longer rated notes.

Verified in code:
- Engine task (0x4200ccc8) and audio output task (0x4200ca8c) form a double-buffered pipeline of 16-sample blocks. Six 16-float bus buffers live in the engine object; output stage multiplies bus 0 by 2^31 with saturating conversion.
- Engine events (24 bytes, queue 0x3fca3960): 0 note on, 1 note off, 2 pitch bend, 3 CC, 4 channel pressure, 5 poly aftertouch, 6 set param, 7/8/9 further param slots, 10 master volume (value/511), 11 LFO number, 12 layer for LED mirror, 13 Fx__resetAll. Types 10 to 13 act only in the sender (Engine__sendEngineEvent). Type 6 also calls Fx__setParam at once.
- Master volume: gain is level squared with a ramp (target, step, s16 counter, ramp length).
- Voice manager: 8 voice slots per part (state 0 free, 1 playing, 2 released, 3 fading). Victim order: free, oldest released, then oldest playing (oldest fading when byte +0x391 is set), else highest note.
- Voice DSP vtable 0x3c070288: +0x10 trigger, +0x14 release, +0x18 finished, +0x1c fast fade, +0x20 setParam, +0x24 render. 32 voice objects at 0x3fcaaf78, stride 0x828.
- Wavetable numbers: UI numbers above 31 select internal n + 24 (internal 32 to 55 are skipped). This explains the skipped tables in the step formatter.
- Shared control tick: 49 samples plus a random -32 to +30 when q > 0.25.
- LFO: shapes 0 sine, 1 triangle, 2 square, 3 falling ramp, 4 rising ramp, 5 noise, 6 sample and hold; skew fixed 0.5; voice copy and global copy differ only in the sine table address.
- Clocked LFO rate = tempo times an entry of the 19 float ratio table at 0x3fcbb478.
- Envelope coefficient table 0x3fccde98: entry i = 1.0 when ROM[i] is above 0x7ffe, else ROM[i] x 0.0625 / 32767.

Corrections vs older notes:
- Event types 11, 12, 13 are now explained on the sender side (see above); the earlier "meaning of 12 and 13 unconfirmed" is resolved.
- LFO LED generator location confirmed (engine+0x1618, writes board+0x25c).
- Wavetable table skip (32 to 55) explained by the +24 rule (previously "inferred user slots").

Rated medium (flagged): Engine NoteProcessor__setParam 0x42013f40, VoiceManager__noteOff 0x40380afc, Voice__trigger 0x4037b61c (filter coefficient formulas not verified numerically; random offset fields not matched to names), Voice__render 0x4037bfb0 (structure read; oscillator interpolation, four filter responses, drive curves and pan law constants not re-derived), TempoClock__advance 0x42016e08 (meaning of doubles at +24 and +48 not checked).

Still unconfirmed:
- Writer of the byte at part+0x391 (steal mode).
- Contents of the 19 entry ratio table (filled at run time, not in the image).
- Glide table semantics and flag polarity.
- Writer of the 5 entry drive table at 0x3c07002c and of the ramp reload lengths.
- Senders of event types 11 to 13 (queue-producer question deferred by the user).

Not part of this group: wavetable cache rows (0x4201682c, 0x4202b5ac, 0x4202b6c4, 0x4202c500), formatter 0x4204090c, boot splash 0x420192cc, and the board, test mode, display driver and ADC rows.

Recheck status after this group: 25 rows still rated notes, 721 verified (strong), 16 medium.

## Round 82 - Recheck group 10: wavetable cache (4 rows)

Rows: 0x4201682c WavetableCache__init, 0x4202c500 WavetableCache__buildAll, 0x4202b6c4 WavetableCache__buildTable, 0x4202b5ac WavetableCache__fillGaps. Together with the already described 0x4202bb70 (synth tables) and 0x4202b7f0 (table 88) this completes the cache.

Verified in code:
- Cache layout (base B = pointer at 0x3fcbdcc0, 0x58c83 bytes in PSRAM): s16 +0 = 100, s16 +2 = 0; 89 tables x 61 waves x 64 bytes from B+4 (table t at B + 4 + t x 3904); flags 89 x 61 bytes at B+0x54d44; user table map 24 x 61 x u16 at B+0x5627a (tables 32..55); 24 bytes at B+0x56dea (use unknown); user wave pool 122 x 64 bytes at B+0x56e02.
- Wave number space used by the table definitions: 0..245 ROM waves, 246..367 user pool (122 slots), 368..421 ROM waves 246..299, 0xffff = empty (filled by averaging), anything else = ROM wave 101.
- ROM table definitions at 0x3c07e240 hold 40 lists (tables 0..27 and 76..87) of 61 x u16.
- Build order in buildAll: user tables 32..55 get no builder (stay empty), 28..31 and 56..75 are code-built, 88 is the special table, the rest come from ROM definitions; fillGaps runs after each.
- Cache waves are unsigned bytes centred on 128 (average in fillGaps is an unsigned sum >> 1).
- Full scan of the code segment for the literal-pool slots of the cache offsets (0x5627a, 0x56e02 and the allocation size): only WavetableCache__init (zeroing) and buildTable read or write them. No code in v1.02 fills the user table map or the user wave pool.
- The only other reader of the cache pointer is WaveSection__draw (display).
- The engine side already uses the same step mapping: UI wavetable numbers above 31 select internal n + 24 (Round 81), so the 24 user tables cannot be reached from the knob.

Corrections vs older notes:
- fcn.4201682c is not only a draw/scan routine: it allocates and initialises the cache (the old line "not a writer" was wrong). The draw code is WaveSection__draw.
- The old statement that the engine mapping was not located is closed by Round 81.

Still unconfirmed:
- Consumer of the two 8192 byte buffers at shared object +28 and +0x2024 (64 rows of 128 signed samples built from table 0 waves 0..60 and table 88 waves 0..2 with a mirrored, inverted second half). Whether the oscillator expands the 64 byte waves the same way is not verified.
- Meaning of the 100 header word and of the 24 bytes at B+0x56dea.
- How an empty (all zero) user table sounds when selected (not reachable from the knob in v1.02).

Recheck status after this group: 21 rows still rated notes, 724 verified (strong), 17 medium.

## Round 83 - Recheck group 11: board, factory tests, display and ADC (19 rows)

All 19 board/test/display/ADC notes rows were disassembled and re-rated. Recheck rows are in cls/map161.txt. No hardware or emulator results were used as evidence.

Verified in code:
- Board__scanInputs (0x42026c48) polls 6 encoders, 15 pots, 16 button ids, and updates 16 LED outputs. Changed controls produce 16-byte UI input events: encoder type 3, pot type 2, button type 0. Board calls the shift-register and debounce helpers and the ADC millivolt converter. The event ring is at 0x3fcd3298.
- Pot__scaleWithHysteresis (0x42027278) uses the 1680 mV breakpoint: <=1680 maps trunc((mV - 10) * 255 / 1670), above maps 255 + floor((mV - 1680) * 256 / 1370), clamped to 0..511. An update requires the raw change to exceed 12 mV and the scaled value to differ. A flag is returned in bit 16.
- BoardAdc__readMillivolts (0x420283d4) selects 15 ADC buffer halfwords in a switch-ordered permutation of 16 slots, then calls adc_cali_raw_to_voltage. BoardAdc__scanTask (0x420284dc) configures continuous ADC sampling at 19200, four patterns, frames up to 256 bytes, rotates GPIO 15/16 mux bits, accepts 12-bit samples on four channel ids and averages 16 samples per logical slot before running 16 smoother records. The board scans 15 physical pot ids even though 16 filter records exist.
- PotSmoother__step (0x420289b8): a = min(1, base + scale*abs(s1 - s2)); s1 += (input - s1)*a; s2 += (s1 - s2)*a. Two float states are stored at +0 and +4. Initializer values used by ADC task are 2.0, 75.0, 0.01. The exact initializer formula feeding base was not decoded.
- ShiftRegister__readButton (0x4204f1e0) confirms the existing active-low 16-id button byte/bit map in Round 19. ShiftRegister__setLed (0x4204f3ac) confirms the 16-id mask mapping into bytes +6 and +7 from the same round. Led__encodeOutput (0x4202722c) uses the LED mode and optional blink timing before packing channel and state.
- Factory-test screens (0x42026238) draw title, separator, optional sublines and flush the frame. The LED test (0x420262b0) turns on all 16 LED records until exit. The pot test (0x4202656c) checks that all 15 scaled positions have hit both endpoint conditions (0 and greater than 510), rather than merely that the pots have been moved. The MIDI test (0x42026748) sends a 3-byte loopback message and checks received bytes. The factory test menu (0x42026910) has six callback-backed entries.
- Display__setBrightness (0x42027360) clamps to index 0..19 and sends commands 0xd9 and 0x81; the latter value is index*16+1 below 15 and 255 from 15 to 19. Display__flushFrame (0x420273f0) queues two 24-bit commands, copies 1024 framebuffer bytes and queues one 8192-bit SPI data transaction. Text__draw (0x42028e0c), Text__drawInverse (0x42028c40) and Text__measureWidth (0x42028ef8) read the font vtable and use the 128x64 one-bit column-major frame.

Corrections / qualifications versus older notes:
- The factory pot endpoint check uses the 0..511 scaled position, not a direct raw ADC reading.
- The ADC acquisition routine constructs 16 smoothing objects, while the front-panel board scan exposes 15 pot ids.
- The LED and button maps are confirmed from the firmware. Physical button label mapping remains unconfirmed by static code alone.

Rated medium (6): FactoryTest__midiTest 0x42026748 (receive frame and timeout interpretation); FactoryTest__runMenu 0x42026910 (exact button navigation and test pointer mapping); Board__scanInputs 0x42026c48 (ring wrap and timing branches containing undecoded SALT/SALTU opcodes); Led__encodeOutput 0x4202722c (mode-2 comparison polarity); BoardAdc__scanTask 0x420284dc (physical mux mapping and some ADC configuration fields); PotSmoother__init 0x42028960 (library math function / derived coefficient formula). These details must not be treated as verified.

Still unconfirmed: exact OLED controller model, physical routing of ADC mux channels to controls, mode-2 LED compare polarity, factory menu navigation mapping and MIDI-test receive framing. Other standing open items and the owner's deferred queue-producer question are unchanged. The next two notes rows are the formatter 0x4204090c and the boot splash 0x420192cc.

Recheck status after this group: 2 rows still rated notes, 737 verified (strong), 23 medium.

## Round 84 - Recheck group 12: formatter and boot splash (2 rows)

Rows: 0x4204090c Format__snprintf and 0x420192cc Boot__showSplash. Both were re-read in full from the Xtensa ELF and re-rated strong. Row file: cls/map162.txt. No hardware or emulator evidence used.

Verified in code:
- Format__snprintf (0x4204090c, 116 bytes) is a bounded printf-style output wrapper. A negative signed size sets errno 139 and returns -1; a nonzero size reserves one character for a trailing NUL. It calls the internal format engine at 0x42041c00, and NUL-terminates for nonzero size. The error rule for format-engine results below -1 is separately covered in the function-map description.
- Boot__showSplash (0x420192cc, 288 bytes) sets LED brightness to 200, forces LED id 1, refreshes shift-register inputs, clears the 128x64 display buffer, draws Protein and a horizontal line at y=22. Startup state 0x3fccf425 becomes 3 for button ids 13, 14 and 15 together; otherwise 1, with Autoplay Mode shown if id 8 is pressed. The normal splash prints Version 1.03 b3 via the formatter and starts the Init task at 0x4201915c.

Still unconfirmed: Physical button labels, the meaning of startup state 3, whether the Autoplay Mode screen affects playback, and why this image labels itself v1.02 while displaying Version 1.03 b3. These are not established by the function alone.

Recheck status after this group: 0 rows still rated notes, 739 verified (strong), 23 medium.

## Round 85 - Classify 19 previously unclassified entry-scan candidates

Scope: 18 old DATA? and 1 U entries, from the existing 2,402-row candidate map. Classification is for coverage, not a new rating pass on class P. Reviewed their disassembly, neighboring routines, callers and referenced literal tables. Code is static evidence only; no hardware, emulator or build work was used. Updated rows are in cls/map163.txt.

- **Non-functions (2):** 0x40378400 is a contiguous IRAM pointer/literal pool (the bytes that resemble `entry` are coincidental). 0x42002b83 is the IROM literal/IEEE-754 constant prefix, also a false prologue. Neither is confirmed executable function code; both are class DATA and excluded from the function total.
- **FreeRTOS/ESP-IDF support (7):** 0x40379710 is a per-core IPC callback task, passed as task entry from 0x42004282. 0x40384e50 is FreeRTOS semaphore/queue waiting (queue+56 count) called from spi_bus_lock_acquire_start and spi_bus_lock_wait_bg_done. 0x40385bf0 is scheduler/task cleanup housekeeping. 0x4038a200 implements TLSF-style allocator size-class/free-list search. 0x4038acd8 performs a byte-sized atomic compare/exchange and is referenced through an IROM pointer by 0x4203d100 and 0x4203d1ff. 0x4038b288 dispatches driver callbacks via a library function table, used by 0x4038b2f0. 0x42052788 is an early CPU/platform startup helper referenced by startup code through a literal at 0x40378498.
- **Memory protection (6):** 0x42005730, 0x42005bcc, 0x42006174, 0x4200621c, 0x420064e0 and 0x420065c4 belong to ESP-IDF's ESP32-S3 memory-permission/PMS register control family. Their literal pools resolve to registers 0x600c10xx, which are in the SoC permission-management region; 0x42006838 and cpu_start 0x42003328 reference these helpers. (The SoC peripheral register address range is cross-checked against the Espressif ESP32-S3 technical reference manual.) The exact per-function mode/bit maps have not all been reconstructed.
- **libm (4):** 0x4203e88c is the single-precision cosine kernel (called by 0x4203da08), 0x4203e944 is the sine kernel (called by 0x4203da08 and 0x4203db20), 0x4203e9dc is a tangent kernel (called by 0x4203db98), and 0x4203ed74 is the IEEE-754 power computation called by the named powf wrapper 0x4203d85c. The last two are only medium-rated for exact mathematical details.

All 17 code candidates are now class L. The seven support-routine assignments and six PMS assignments are library classifications, not claims that every register or exceptional branch is fully understood. Exact identity of 0x4038b288's driver subsystem, several FreeRTOS-private routines, and the PMS selector enums remain unconfirmed. Do not silently upgrade these medium rows to strong. No new class P function was found.

UF2 check: 200-byte sequences sampled from each of ELF .iram, .iram1, .irom, .drom and .dram matched payloads assembled from PROTEIN.UF2; this supports using the supplied ELF for this audit, but is not an independent full byte-for-byte partition/layout verification.

Current coverage: 2,402 entry-scan candidates = 762 Protein functions (739 strong, 23 medium), 1,638 library functions (several newly classified at medium confidence), and 2 explicitly identified non-function data spans. **0 unclassified entry candidates remain.** The heuristic prologue scan is not proof that every possible function entry has been recovered. Old open items in the handoff remain open, including the owner's deferred queue-producer question. No build work has begun.

Recheck status after this group: 0 Protein rows rated notes, 739 verified (strong), 23 medium. Next suggested group: the 23 medium-rated Protein functions, starting with voice triggering/rendering and note-off.

## Round 86 - Medium-rating recheck: voice / note-off

Rows: 0x4037b61c, 0x4037bfb0, 0x40380afc. Class P, static ELF evidence only; no hardware, emulator, or firmware build work. Row file: cls/map164.txt.

- VoiceManager__noteOff (0x40380afc) upgraded to strong: the eight-slot search, released and pending-slot paths, sustain held table, saved release timestamps, and mono return to the highest held different note are all traced. Release invokes the voice DSP vtable at +20; the replacement note invokes the part note-on vtable.
- Voice__trigger (0x4037b61c) stays medium: randomized offsets and filter coefficient arithmetic are not fully matched to DSP parameter semantics.
- Voice__render (0x4037bfb0) stays medium: oscillator interpolation, drive and filter responses, and exact panning constants are not fully reconstructed.

Recheck status after this group: 0 rows still rated notes, 740 verified (strong), 22 medium.

## Round 87 - Medium-rating recheck: chorus, delay, phaser, reverb and tremolo process

Rows: 0x4200feb4, 0x42010ba0, 0x42012b18, 0x42013420, 0x42013798. Class P, static ELF evidence only; no hardware, emulator, or firmware build work. Row file: cls/map165.txt.

- All five process functions remain medium. The block structure and existing parameter/phase/ramp descriptions remain useful, but do not establish the exact DSP response for every branch.
- Chorus: high-/low-cut exponential factor and compensation interpretation unresolved. Delay: feedback and interpolation coefficients unresolved. Phaser: per-section model frequency/coefficients unresolved. Reverb wrapper: piecewise Time/Color mapping unresolved. Tremolo: tempo-sync arithmetic and filter coefficient factor unresolved.

Recheck status after this group: 0 rows still rated notes, 740 verified (strong), 22 medium.

## Round 88 - Medium-rating recheck: parameter, cache, clock, text and persisted-value handling

Rows: 0x42013f40, 0x4201682c, 0x42016e08, 0x42024708, 0x420294f0. Class P, static ELF evidence only; no hardware, emulator, or firmware build work. Row file: cls/map166.txt.

- TextEntry__onEncoder (0x42024708) upgraded to strong. Its two modes, 78-position character lookup, cursor limits, append-space behavior, +168/+169 flag writes, and redraw request follow directly from the code. The two flag fields are described by offsets, without speculative UI names.
- NoteProcessor__setParam remains medium due to unlabelled step-record ids. WavetableCache__init remains medium because scratch-buffer consumers and some field meanings are unknown. TempoClock__advance remains medium because carry encoding and double-field semantics are not fully verified. Sound__migrateValue remains medium pending a path-by-path audit of older-format conditional branches.
- No change to the owner-deferred queue-producer question or other unrelated open issues.

Recheck status after this group: 0 rows still rated notes, 741 verified (strong), 21 medium.

## Round 89 - Medium-rating recheck: factory tests, front-panel scan and ADC

Rows: 0x42026748, 0x42026910, 0x42026c48, 0x4202722c, 0x420284dc, 0x42028960. Class P, static ELF evidence only; no hardware, emulator, or firmware build work. Row file: cls/map167.txt.

- PotSmoother__init (0x42028960) upgraded to strong. For constructor inputs x=a3, y=a4, z=a5, base coefficient = 2*tan(pi*x/y)/(1+tan(pi*x/y)); scale = 4*z. The tangent wrapper 0x4203db98 and kernel 0x4203e9dc establish the previously unknown library call. The ADC caller passes x=2, y=75, z=0.01, giving base approximately 0.15493448 before float rounding.
- Factory MIDI receive framing/timeouts, factory menu callbacks and physical navigation, Board__scanInputs ring/timing compare opcodes, LED mode-2 compare polarity, and physical ADC mux connections remain unconfirmed. Those five rows remain medium.

Recheck status after this group: 0 rows still rated notes, 742 verified (strong), 20 medium.

## Round 90 - Medium-rating recheck: reverb core

Rows: 0x4202aa9c, 0x4202ace0, 0x4202ad50, 0x4202b018. Class P, static ELF evidence only; no hardware, emulator, or firmware build work. Row file: cls/map168.txt.

- ReverbCore__updateTapGains (0x4202ace0) upgraded to strong. The cutoff is v > 0.5, with g from core+0x18 and v from core+0x20. Otherwise u=2*v, g1=g*(5-4*u), g2=u*g1, g3=u*g2; the signed seven-value sequence g1,g2,-g2,g3,-g3,-g3,-g3 is stored in both early-tap groups.
- ReverbCore__diffuse remains medium because all-pass and delay-line arithmetic is not fully characterized. ReverbCore__init remains medium because individual stored coefficients are not all semantically identified. ReverbCore__process remains medium because pre-delay, diffusion and mixing calculations still need term-by-term analysis. Exact musical meaning of gain inputs g/v is not asserted.

Recheck status after this group: 0 rows still rated notes, 743 verified (strong), 19 medium.

## Round 91 - Voice DSP arithmetic: random routing, waveform sampling and pan

Rows: Voice__trigger 0x4037b61c and Voice__render 0x4037bfb0; `cls/map169.txt`. Static ESP32-S3 ELF disassembly only, no hardware, emulator or firmware modifications. Both rows remain medium. This is a focused recheck of unresolved mathematical paths, not a claim that all render branches are proven.

Verified in code:
- Voice__trigger draws three independent `rand()` samples and reduces each modulo 1000. They are multiplied by q from the per-part hook (zero on the null-pointer path). The first has coefficient 0.0252 and writes voice+0x28; the second and third have coefficient 0.0001 and write voice+0x2c and +0x30. The computed magnitudes are therefore [0, 24.9748*q], [0, 0.0999*q], and [0, 0.0999*q] for nonnegative q (float rounding aside). The fixed scaling factors are stored as IEEE-754 floats 0x3cce703b and 0x38d1b717.
- In Voice__render, voice+0x28 is added to the wavetable scan position for oscillator 1 (0x4037cbfb) and oscillator 2 (0x4037ccb4); voice+0x2c is added to filter cutoff (0x4037dae9); voice+0x30 enters oscillator 1 pitch computation (0x4037cdaf). These names describe where values are consumed, not the name of a UI control. Earlier uncertainty about which destination each random field serves is resolved.
- Two per-oscillator waveform decoders copy raw unsigned cache bytes into 128 signed playback samples: for each 64-byte halfwave, first half is `source[k] - 128`; mirrored second half is `127 - source[63-k]`. The oscillator then selects one signed sample via the top seven bits of the 32-bit phase accumulator (`phase >> 25`), with no adjacent-sample interpolation (e.g. 0x4037d494-0x4037d4d2). This confirms the halfwave expansion already proposed in Round 7; it does not identify the unrelated two 8192-byte scratch-buffer consumers from the wavetable cache init.
- The final pan calculation (0x4037e50c-0x4037e55f) clamps the effective pan to 0..1 and constructs the two complementary quadratic coefficients `p*(2-p)` and `(1-p)*(1+p)`, which both equal 0.75 at center. The effective pan combines a smoothed parameter at voice+0x4e0, a mod-matrix destination float and voice+0x4f4. This is not a sine/cosine equal-power law. Further envelope/gain scaling happens before output stores.

Corrections to older descriptions:
- The render row's vague claim that oscillator samples were "interpolated" is corrected: oscillator readout indexes the 128-entry signed waveform directly, without adjacent-sample interpolation. Wavetable scan positions can change which discrete wave is selected, but this is distinct from interpolating sample points.
- The old phrase "cascaded biquad stages" was too specific. The multiple recursive filter structures are not yet proved as conventional biquads, so the row no longer asserts this.

Still unconfirmed:
- Full coefficient derivation and transfer functions for filter modes 0..4; mode-dependent signal path and saturation feedback.
- Full drive-type crossfade/response terms in Voice__render, the remaining state/control paths, and whether all pan/gain branches have identical normalization.
- The per-part q source remains unconfirmed as active in normal operation (earlier Round 81 evidence suggests a null hook). The three random samples are read even when q is zero. No claim of audible randomness is made.
- The separate wavetable cache scratch buffers at shared DSP object +28 and +0x2024 are still not associated with an audio consumer.

Recheck status after this group: 0 rows still rated notes, 743 verified (strong), 19 medium.

## Round 92 - Voice filter transfer structures and drive amount arithmetic

Rows: Voice__trigger 0x4037b61c, Voice__render 0x4037bfb0. Row file: cls/map170.txt. Static ELF and Xtensa disassembly only, no hardware, emulator, or firmware modifications. Neither row is upgraded to strong.

Verified:
- Filter type at voice+0x4ac dispatches 0=LP24, 1=LP12, 2=HP24, 3=HP12 and 4=Drive, and values outside the 0..4 range take an unfiltered unity-gain path. In mode 4 the waveshaper call is to 0x42011768. The filter/resonance values are clamped to 0..1 before mode selection (0x4037db2a-0x4037db42).
- Both LP and HP modes run four recursive stages on a 16-float sample buffer. The two 12 dB modes take the stage 2 result even though stages 3 and 4 also run. The two 24 dB modes use the stage 4 result. LP modes apply a final factor 0.75 (0x4037e385), and HP modes apply 1.2 (0x4037e4a7). The literal note LP24/LP12/HP24/HP12 is a structural mode name, not a hardware spectrum measurement.
- LP stage algebra in 0x4037e291-0x4037e380: each stage forms delta=g*(input - old_state), emits y=old_state+delta and stores new_state=old_state+2*delta. It feeds y to the next LP stage, with the 4-stage path using the final y and the 2-stage path storing the second y. The resonance feedback is subtracted from input before saturation. This agrees with the earlier Round 32 structural description.
- HP stage algebra in 0x4037e3a4-0x4037e4a7: each stage forms delta=a*(input - old_state), advances state by 2*G*delta and feeds delta to the next HP stage; a is a reciprocal-type coefficient approximately 1/(1+G). The resonant HP feedback enters with an addition before saturation. Mode 2 selects fourth delta and mode 3 selects second delta. These are code recurrence descriptions, not a frequency-response claim.
- LP cutoff baseline G_LP = 0.000582427*2^(1.66667 + 10.4167*c); HP baseline G_HP = 0.000582427*2^(12.0833*c). These are per-stage coefficient *inputs*, not a verified direct frequency-in-Hz formula. Stage factors and feedback further alter the effective response. The exp2 approximation uses a polynomial after exponent splitting, followed by approximate inverse arithmetic using float-bit manipulation and Newton refinement, so expressions written as reciprocals here identify the target mathematical operation rather than claiming bit-exact division. LP mode 1 scales resonance by 0.8, HP mode 3 by 0.85 (code at 0x4037db64 and 0x4037de5c).
- In mode 4 the control update code at 0x4037e11c-0x4037e13a assigns state (voice+0x3d4)+4 = cutoff*c_scale = c/2 and post gain at voice+0x3d0 = (1-0.7*c)^2. The render calls the 16-sample Drive__shapeChannel wrapper 0x42011768 with the shape state voice+0x3d4. Drive__shape 0x42011584 applies pre-gain 128*d^2 (d=c/2). The wrapper 0x42011768 tests d >= 0.125, and below it adds dry_weight*input to the shaped result, dry_weight=(0.125-d)*8 = 1-4*c. This is additive blending, not a normalized crossfade; above the threshold no wrapper dry sample is added. The post gain multiplies the result in the subsequent output path.
- VoiceDsp__setParam 0x4037ac44 for k=52 clamps the DriveType integer to 0..4, stores that type at voice+0x3e0 and reads a float into voice+0x3d4 from 0x3c07002c. All five DROM table values are 0.200000003 (IEEE-754 single precision). DROM is an immutable image region; the previously open request for a *runtime writer of this ROM table* is unnecessary. This does not resolve the writers of ramp reload lengths or UI label-to-type mapping.

Corrections to previous notes:
- The wrapper condition depends on the *unsquared* amount d stored in state+4. Only the core waveshaper squares d to calculate input pre-gain. Earlier notes calling state+4 a squared amount were inaccurate.
- The drive path has a verified dry blending threshold and explicit output attenuation, so describing its mixing as wholly unknown is outdated.

Still unconfirmed: full mapping of every precomputed resonance/feedback weight and exact nonlinearity/normalization under every mode; precision of approximate division/exp2 relative to ideal mathematical formulas; complete trigger/retrigger coefficient branches and other render control logic; correspondence of the five DriveType index values to the display labels; physical filter frequency response. No unsupported rating upgrades.

Recheck status after this group: 0 rows still rated notes, 743 verified (strong), 19 medium.

## Round 93 - Effects DSP medium recheck: chorus, delay, phaser, reverb and tremolo

Rows: 0x4200feb4, 0x42010ba0, 0x42012b18, 0x42013420, 0x42013798. Row file: cls/map171.txt. Static firmware ELF disassembly only; no hardware test, emulator or firmware modification. All five remain medium, not because they are unmapped but because remaining coefficient meanings or optional paths are not fully proven.

Verified corrections and progress:
- Chorus, Delay and Tremolo cutoff/smoothing exponent: the apparent extra 0.5 factor from Round 77/87 is not part of the final exponent. In each coefficient computation an x^3 value is scaled by float 0.5 and then passed through double multiplication by -2*pi, giving expf(-pi*x^3), not expf(-0.5*pi*x^3). Double constant -2*pi is at literal words 0x42000ca4/0x42000ca8, passed to __muldf3 and then __truncdfsf2 before expf at 0x4203d800. For Tremolo, x = 0.06 + 0.3*(1-Smooth); both filter branches reuse the same input exponent (0x420139e7-0x42013a59). For Chorus and Delay, x = 1-HighCut or 1-LowCut and their f2/f1 multiplier paths are 0.5 (Chorus 0x42010084, 0x420101bf, 0x420101ea-0x4201022a; Delay 0x42010ea0-0x42010f43). Existing rows erroneously inserted a second 0.5 after converting the double constant.
- Phaser Model 0 uses table A at 0x3fca5014 and stateful 0.9 / 0.1 updates, with one 0.008 correction, to produce two coefficients expf(-pi*computed_value) sent to Allpass2__setCoeffs (0x42012fe8-0x42013081). Model 1 uses table B at 0x3fca541c and writes coefficients directly. The full section-by-section frequency and all-pass response for both models remain unresolved.
- Reverb__process Color branch (0x4201355a-0x42013639) is now distinguished from Time. Color is object +0x408, optional modulation pointer +0x424. With z = 2*clamp(Color+mod,0,1)-1, if z<=0 the code computes q=-0.99-1.89*z and stores [q,(1-q)/2,(1-q)/2] at +0x3d0,+0x3d4,+0x3d8; if z>0 it computes q=0.99-1.69*min(0.3*z*z,1) and stores [q,(1+q)/2,-(1+q)/2]. The Time value comes from object +0x400, optional modulation pointer +0x420 and a stack float. It is clamped and passed separately to ReverbCore, stored at +0x3e4 and +0x30 (0x420135f3-0x42013708). The older row incorrectly called the piecewise coefficient law a Time law and assigned Time the Color modulation pointer. Related 16-byte coefficient records and ReverbCore processing still need checking.
- Tremolo sync branch: when +0x74 >= 0.1, a nonnull +0x84 pointer enables selection of one of 32 ratio doubles by clamping int(float at +0x78 +0.05) to 0..31. The ratio is multiplied by the double read from pointer+24, and stored phase step +0x1c is 1/max(1,ratio*double). A null pointer keeps the old phase step. The +0x88 pointer gates an additional double-based phase offset correction; exact source fields and intended meaning are not confirmed (0x42013884-0x420139c9).

Still unconfirmed (retain medium): Chorus compensation byte +0x200 and exact behavior of optional modulation pointers; Delay stereo cross-feed/output tap routing and feedback/interpolation handling under all flags; Phaser section-spacing, Model 1 coefficient semantics and complete cascaded response; Reverb filter-record fields and ReverbCore network response; Tremolo secondary sync phase offset and hardware/runtime activation of sync pointers. No ratings were upgraded without full function-level proof.

Recheck status after this group: 0 rows still rated notes, 743 verified (strong), 19 medium.

## Round 94 - ReverbCore diffuse, init and process recheck

Rows: 0x4202aa9c ReverbCore__diffuse, 0x4202ad50 ReverbCore__init, 0x4202b018 ReverbCore__process. Row file: cls/map172.txt. Static ELF disassembly only, no hardware, emulator or firmware modification. Corrected descriptions and upgraded diffuse and process to strong; init remains medium.

Verified against instructions:
- ReverbCore__diffuse (0x4202aa9c) is called from 0x4202b4f0 with a2=core+0x28, a3=ring pointer core+4, a4=uint16 index core+8, and a5=float core+0x20. The older description's a5 as an audio input was wrong: it is a coefficient used by later stages. The routine does not return a rendered audio sample.
- All-pass section (0x4202aabc-0x4202ab04): for a 16-byte record with uint16 endpoints A at +0 and B at +4 and gains g at +8, h at +12, let i=(write-B+k)&65535, j=(write-A+k)&65535 and old x=ring[i], y=ring[j]. It stores ring[i]=g*y+h*x and ring[j]=y-g*x for each k=0..15. Eight records are initialised (core+0x28 count=8), but execution uses the runtime count.
- Modulated delay section (0x4202ab1d-0x4202ac06): count at diffuser+0x144; 36-byte records, rate state at record+0, target at +4, phase at +8. Rate moves toward target by 0.01 per block. Phase increments by (rate/44100)*16 with wrap at 1, and the triangular phase multiplier is 2p for p<0.5, else 2-2p. Depth at record-12 moves toward target at record-8 by 0.01. An unsigned integer and fractional part of triangular modulation times depth determine a 16-sample ring relocation via 2-point linear interpolation. The code stores the resulting 16 floats back in the shared ring.
- Intermediate diffusion (0x4202ac18-0x4202ac74): count at diffuser+0x1d8, stride 16, per-record uint16 delay at +0x1dc and float coefficients at +0x1e0,+0x1e4,+0x1e8. For each of 16 samples writes ring[next]=c0*ring[previous]+a5*c1*ring[next]+a5*c2*ring[nextnext], using the pre-write values. The final delayed sum stage (0x4202ac79-0x4202acc5) reads three delay offsets per 12-byte record (from diffuser+0x220) and writes ring[write-third+k]=ring[write-middle+k]+a5*ring[write-first+k], with uint16 wrap. Init sets two records for each of these last two stages.
- ReverbCore__init (0x4202ad50-0x4202aef1): +0 is a read-only dispatch pointer, +4 a null ring pointer, +8 contains PACKED two 16-bit indices (0,16), value 0x00100000. Earlier text incorrectly said the +8 word is a second pointer. Counts are 8 at core+0x28, 2 at +0x16c, 2 at +0x200, 2 at +0x244, 7 left at +0x278 and 7 right at +0x27c. Hardcoded delay values, coefficients, state defaults and tap offsets are written; the gain updater 0x4202ace0 completes the tap gain entries. +0x3f0 clear counter is zero initially.
- ReverbCore__process (0x4202b018-0x4202b50e) uses a 16-sample block. The positive +0x3f0 clear-counter path multiplies current gains +0x18/+0x20 by 0.92, updates tap gains, clears a ring segment, copies the two dry inputs to distinct outputs, then mixes taps and increments both uint16 indices by 16, without applying the diffuser.
- On the normal path (0x4202b175-0x4202b1f2), +0x20 follows target +0x24 and +0x18 follows target +0x1c; when |delta|>0.01 the new current value is current+0.01*delta, otherwise it snaps if nonzero. The tap gains are recalculated if +0x18 changes. The input streams are summed sample by sample into the ring at current index core+10 (0x4202b1f5-0x4202b2bb). Dry outputs are copied unchanged if +0x18 <0.45; for +0x18 >=0.45 they are scaled by max(0,10-20*(+0x18)) (0x4202b2be-0x4202b494). Ring samples then receive a 3-tap filter with base offset +0x3c0 and floats at +0x3c4/+0x3c8/+0x3cc (0x4202b494-0x4202b4e8); processing calls diffuser with a5=core+0x20, mixes early-reflection taps and increments both circular indices by 16. No claims are made about measured audio quality or spectral response.

Corrections to older notes:
- Diffuser float argument is a gain/coefficient from core+0x20, not a fresh input sample. The incoming audio is prewritten to the ring by ReverbCore__process.
- Initialisation word +8 is 0x00100000, containing index 0 at +8 and 16 at +10, not a pointer. +0 is the read-only dispatch pointer; +4 is the ring pointer.
- The circular-buffer and dry-output paths of ReverbCore__process are now traced and no longer described as only structural.

Still unconfirmed: the specific intent behind all chosen initial delay length constants and miscellaneous initial state fields (notably +0x3e0) in ReverbCore__init; precise psychoacoustic or transfer-function interpretation of the complete reverb network remains outside static verification. Reverb__process wrapper remains medium; no hardware or emulator evidence. No firmware modification.

Recheck status after this group: 0 rows still rated notes, 745 verified (strong), 17 medium.

## Round 95 - Reverb wrapper recheck against verified ReverbCore

Row: 0x42013420 Reverb__process (752 bytes). Row file: cls/map173.txt. Reviewed full Xtensa wrapper and direct set-pre-delay callee 0x4202aef4, cross-referenced strong-rated ReverbCore__process (0x4202b018) and ReverbCore__diffuse (0x4202aa9c) from Round 94. Static ELF evidence only; no firmware changes, emulation or hardware test. Upgraded Reverb__process from medium to strong.

Verified in code:
- Pointer adjustment is essential: wrapper outer object O=a2, inner core C=O+0x0c. The wrapper passes a10=O+0x0c as ReverbCore a2. O+0x20c therefore means core count C+0x200, not a floating-point coefficient. Per-section writes beginning at O+0x214 map to the first 16-byte diffuser record's coefficients C+0x208, C+0x20c and C+0x210. Both the core's own 3-tap ring filter C+0x3c4/+0x3c8/+0x3cc and each diffuser damping record receive the same piecewise Color coefficients.
- With Color c clamped 0..1 and z=2c-1, if z<=0, q=-0.99-1.89*z, b1=b2=(1-q)/2. Otherwise q=0.99-1.69*min(0.3*z*z,1), b1=(1+q)/2, b2=-b1. The normal wrapper writes q,b1,b2 to O+0x3d0/+0x3d4/+0x3d8 and loops over count C+0x200, writing three coefficients at diffuser-record offsets +4,+8,+12. The core ring filter consumes the former (C+0x3c4/+0x3c8/+0x3cc), and ReverbCore__diffuse consumes the latter, multiplying the two outer diffuser coefficients by its Time-gain argument.
- DryWet m=clamp(O+0x404 + float argument entry a7 + optional *pointer at O+0x41c,0,1). Cache O+0x3dc=m^2, set core level target C+0x1c=m^2/2 (core smooths target into C+0x18). Color optional modulation pointer is O+0x424. Time t=clamp(O+0x400 + extra float on caller stack at a1+64 + optional *pointer O+0x420,0,1), cache at O+0x3e4 and set core feedback target C+0x24=t (core smooths into C+0x20 and uses it for diffusion, delayed-sum feedback and seven-tap gains). The three modulation pointers are distinct and the Round 93 correction stands.
- If pending update byte O+0x428 is nonzero and the core clear counter C+0x3f0 is zero, copy O+0x40c (PreDelay float) to O+0x3e0 and call 0x4202aef4 with C and that float. For normalized PreDelay x, setPreDelay computes f=16370*(x+x*x), truncates and caps to 32736, rounds down to 16-sample granularity, and sets 16-bit write index C+0x0a = (read index C+0x08 + 16 + rounded f) mod 65536. This corrects the older shorthand in Round 73 that omitted the leading x in the x*(1+x) term.
- In the same update branch, modulation-depth targets at C+0x17c and each next +36 bytes get corresponding baseDepth*(5*ModDepth O+0x410), reading baseDepth C+0x174 and stride +36; rate targets at C+0x188 and next +36 get baseRate*(20*(ModRate O+0x414)^3), from baseRate C+0x180 and stride +36. Both use the 36-byte modulated delay record count at C+0x16c. Stores auxiliary float f=O+0x418 to O+0x3f8 and C+0x14=2*f, C+0x10=2-2*f; clears pending flag only after this update. If clear counter is positive, update is deferred and the pending byte is preserved.
- Calls the already-verified ReverbCore__process on every block, even when the update branch is skipped. No SALT/SALTU instruction gaps in this 752-byte wrapper or the 0x4202aef4 callee.

Corrections to earlier reverb notes:
- The per-section coefficient writes O+0x214 etc. are C+0x208 etc., within the two C+0x200-count diffuser records, not unrelated/unidentified 16-byte records. All exact coefficient roles in the wrapper now have a verified destination in the core.
- The normal dry/wet scalar target is C+0x1c (O+0x28) and the Time target is C+0x24 (O+0x30); the older vague terms 'passed at' cannot hide the O versus C offset difference.

Still unconfirmed: runtime origins and semantics of the three optional modulation-pointer values, the audible frequency response and precise acoustic intent of hardcoded delay constants, and nonfinite or out-of-range parameter behavior. These are not necessary claims in the function description. ReverbCore__init stays medium; the owner's other open items and deferred queue question remain unchanged. No firmware modification.

Recheck status after this group: 0 rows still rated notes, 746 verified (strong), 16 medium.

## Round 96 - Chorus and Delay DSP recheck

Rows: 0x4200feb4 Chorus__process (2692 bytes), 0x42010ba0 Delay__process (2012 bytes); file cls/map174.txt. Revisited all control-flow branches and arithmetic in the original Xtensa ELF. No firmware modification, emulator, hardware test, or simulated observations. Both functions upgraded medium to strong for the instruction-level claims in their revised descriptions.

Verified in code:
- Chorus__process uses bounded depth, speed, center, feedback and dry/wet controllers with nonnull optional modulation pointers at +0x48, +0x4c, +0x50 and +0x44 respectively. Four LFO offsets and 1..4 voices per channel feed interpolated ring delays. Shape 0 triangle and shape 1 sine use distinct paths, and the per-block delay-target ramps advance per output sample. Filter exponent remains expf(-pi*x^3) from Round 93. All 16 left and 16 right outputs are multiplied by the final compensation gain when it differs from 1.0 (0x4201087f-0x4201092b). Reset sets +0x200 to 1 (0x4200fe48); process always clears +0x200 to 0 on the common return at 0x42010875-0x4201087d, whether gain is unity or not. Older notes asserting the byte is cleared only on the unity path were incorrect. No external meaning for the byte is asserted.
- Delay__process uses separate control words for Feedback (+0x30) and stereo cross-coupling (+0x34); they are not the same knob. Let x=+0x34, A=(1-x)^4, B=1-A, H=B/2, p=clamp(Feedback+stack+optional,0,1)^2, uL/uR=raw block input samples and yL/yR=post-filter interpolated delay taps. The sample written into the left delay ring is (1-H)*uL+H*uR+p*(A*yL+B*yR). The right ring receives A*uR+p*(B*yL+A*yR), without a direct uL term. Code evidence: 0x42011244-0x420112f7. Both ring writes clamp to +/-100.
- Delay tap interpolation, confirmed at 0x420110ac-0x420110d7 (left) and 0x420111c4-0x420111f3 (right), is frac*older+(1-frac)*newer at write-trunc(delay)-1 and write-trunc(delay), modulo 441000. The slewed delays move by 0.000125 of the target difference per sample, clipped to [0,440999] (0x42010f86-0x42011016).
- The DC blocking calculation before the two one-pole filters uses tap-prevTap+0.99*prevOutput, and the filter chain computes highcutLP then highcutLP-lowcutLP on each side (0x420110f5-0x42011163 and 0x42011214-0x42011280). The shared HighCut/LowCut exponent correction from Round 93 remains expf(-pi*x^3), NOT expf(-0.5*pi*x^3). Positive and negative infinity are explicitly detected and zeroed for interpolated taps and DC outputs; those tests do not explicitly catch NaN. An older blanket claim that all non-finite values are zeroed was too broad.
- Delay output remains separate from buffer cross-feedback: with m=clamp(DryWet + a7 + optional,0,1), dry=1-m*m, wet=2*m-m*m. Cached gains at +0x100/+0x104 advance by 1/16 of the target difference per sample, except the reset flag +0x108 snaps them to target; output L=dry*uL+wet*yL and output R=dry*uR+wet*yR (0x42010e25-0x42010e88, 0x42011016-0x4201103c, 0x420112fa-0x4201135f). Ring clear bytes +0x84/+0xa8 and end-of-block reset flag are accounted for.

Corrections to older notes: Chorus +0x200 clears on both compensated and uncompensated returns; Delay stereo mix and output equations are now explicit; Delay tap clipping max is 440999, not generically 441000; explicit infinity tests are not full non-finite/NaN sanitization.

Still unconfirmed outside the proven function-level logic: the broader role/consumer of Chorus byte +0x200, writer of the Delay +0x40 offset word and physical/clock unit semantics of its +0xfc pointer, and measured acoustic response. The code-level math of the two functions is no longer counted medium on account of those external items. Other previously tracked open topics and the user's deferred queue-producer question are unchanged. No firmware modification.

Recheck status after this group: 0 rows still rated notes, 748 verified (strong), 14 medium.

## Round 97 - Phaser and Tremolo DSP recheck

Rows: Phaser__process 0x42012b18 (1956 bytes), Tremolo__process 0x42013798 (1300 bytes), plus correction to the already-strong Tremolo__ctor row 0x42013cac. File cls/map175.txt; supporting disassembly effects_round97/*.txt. Full Xtensa control-flow bodies, Phaser__tableLookup 0x42012ac4, Allpass2__setCoeffs 0x4202a2b0, Tremolo__init 0x42013d34 and Part__buildEffects 0x4200f68c rechecked. Original ELF only; no emulation, hardware or firmware modifications. Both DSP functions upgraded from medium to strong for instruction-level behavior described.

Verified Phaser details:
- For Model 0 (Nave) the two 256-point table A lookups feed 0.9*old+0.1*new frequency smoothing at +0x54c (left) and +0x550 (right). Radius is expf(-pi*(0.008+0.1*Spread)) and both first sections receive Allpass2__setCoeffs with that radius and channel-specific frequency (0x42012fe8-0x42013081). Earlier notes left this frequency/radius split ambiguous.
- For Model 1 (PPG), let k be channel table B lookup and h be clamped Spread. Define u=(2-k)*h*k, C=2-k*k-u, D=u-1. Both first-section 5-float coefficient arrays are [C,D,D,C,h] (0x42012f2f-0x42012fbc). The initial coefficients differ by channel lookup k.
- For either model, the first section's five coefficients are replicated to the remaining active 40-byte sections for that channel (0x4201309c-0x42013120). Allpass sample recurrence using coefficients c0..c4 and states x1,x2,y1,y2 is y=c2*x+c3*x1+c4*x2+c0*y1+c1*y2, then shifts two input and two output history samples (0x420131a1-0x42013278). This explicitly establishes Model 1's actual recurrence without assuming a conventional ideal all-pass transfer under arbitrary coefficients. Stages index uses [1,2,4,6,8].
- Gain targets, feedback mixing, phase-shifted LFO, 16-sample input fade on reset, output dry/wet mix and explicit NaN/positive-negative infinity sanitization are checked in 0x42012b18-0x420132b9. No claims about measured tonal response or exact external modulation sources.

Verified Tremolo details and correction:
- Tremolo__init (0x42013d34) writes actual nonzero pointers +0x84 and +0x88 when invoked by Part__buildEffects (0x4200f7d7-0x4200f7e3). Constructor initializes them to zero first, but the older claim that no writer exists was false. The strong-rated constructor row is corrected. This does not establish that the tempo-sync selector +0x74 is enabled.
- Free-rate phase step is [0.1+39.9*s*s]/44100 for clamped speed s (0x42013815-0x42013881). When selector +0x74>=0.1, a nonnull +0x84 pointer selects a 32-entry double beat-ratio (index trunc(float +0x78 +0.05), clamped 0..31), multiplies by the double at the pointer+24, truncates product to float duration, and stores phase step =1/max(1,duration) (0x42013884-0x42013940). If +0x84 is null, the old step persists.
- The optional +0x88 reference object provides gate byte +0x1435 and a nested pointer at +0x13e0. If the gate is nonzero, nested double R is read from nested+40 (or R=0 if nested pointer null), T from primary +0x84 pointer+48. With Q the saved selected ratio double (not the ratio times primary+24 duration) and float offset p at +0x70, it computes z=float((T-R)/Q+p), stores z-trunc(z) in left phase +0x14, then proceeds. Negative z retains a negative fractional remainder due truncation, so the expression must not be replaced by floor-based frac (0x42013948-0x420139c9). The nested clock's functional name and whether the gate is set during normal operation are unconfirmed.
- All three LFO outputs (sine with interpolation for every shape other than 1 or 2; triangle for 1; square plus one-pole smoothing for 2), phase wrapping, smoothing exponent expf(-pi*x^3), and squared stereo output gain were traced to the 0x42013ca9 return. There is no missing control-flow or unknown Xtensa SALT/SALTU opcode in either function.

Corrections to earlier notes: Model 0 Phaser exponent input is 0.008+0.1*Spread, while channel-specific frequency comes from smoothed table A; Model 1's exact 5 coefficients and per-section replication replace vague frequency/spacing descriptions. Tremolo's two clock pointers are written in init, contrary to older notes. The identity of the external time-state fields is not inferred, and no claim of audible behavior or hardware verification is made.

Still unconfirmed: what external runtime configuration selects Tremolo tempo sync (+0x74), meaning of nested clock state/gate, identities of optional effect parameter sources, physical response and several other handoff topics. These are separate from the proven instructions. Other unconfirmed items and the owner's deferred queue-producer question unchanged.

Recheck status after this group: 0 rows still rated notes, 750 verified (strong), 12 medium.

## Round 98 - Final sweep of 12 medium-rated Protein functions

Rows: 0x4037b61c, 0x4037bfb0, 0x42013f40, 0x4201682c, 0x42016e08, 0x42026748, 0x42026910, 0x42026c48, 0x4202722c, 0x420284dc, 0x420294f0, 0x4202ad50. The revised rows are in cls/map176.txt. Ten functions upgraded to strong after reviewing their previously unverified branches and data references; both large voice-DSP functions remain medium. All findings derive only from static reading of the original ELF. No firmware modifications or hardware/emulator tests.

New verified findings (function-level):
- TempoClock__advance (0x42016e08): the previously unrecognized Xtensa opcode at 0x42016e11 is SALTU, yielding a true unsigned carry from the low-word addition. It increments the 64-bit sample counter, divides the double-converted increment by the samples-per-beat double at +24/+28 when >5, and adds the result to the beat accumulator +48/+52. TempoClock__init (0x42016b40) sets the rate to 2646000/BPM, independently establishing beats as the accumulator unit. Rated strong.
- Led__encodeOutput (0x4202722c): the undecoded opcode at 0x4202726a is SALTU, making mode-2 state equal to (period/2 integer) < (elapsedMs % period) with a *strict* comparison. This lights the second part of the period at the logical-LED level; actual physical LED polarity is not established. Rated strong.
- Board__scanInputs (0x42026c48): the four ring-index comparisons 0x42026cda, 0x42026db0, 0x42026e57 and 0x42026efc are SALT/SALTU, verifying bounded wraparound. The comparison at 0x42026d39 is SALTU; encoder scan occurs before the full-scan timer, and the pot/button/LED scan is skipped while unsigned elapsed milliseconds <=9. The event ring records are 16 bytes, with types 3 for encoders, 2 for pots, and 0 for buttons. Rated strong, without claiming physical wiring.
- FactoryTest__runMenu (0x42026910): six immutable callback tables name the MIDI, Pot, Encoder, Button, LED and Audio tests. Their entrypoint addresses are respectively 0x42026748, 0x4202656c, 0x42026644, 0x420264bc, 0x420262b0, and 0x4202630c. The menu moves via *encoder 0*, input event type 3 with id 0 and its signed delta, not via a button-type event as older notes claimed. Rated strong.
- FactoryTest__midiTest (0x42026748): transmits the MIDI triple 0x90,0x41,0x6f; polls with a 1001-iteration budget and 1ms unsuccessful sleeps; dequeues a 12-byte UART record from ring pointer +0x274 and read index +0x27c, with occupancy at +0x284, copying 11 bytes; returns pass on exact three-byte match and failure on mismatch/timeout. No external loopback cable wiring is claimed. Rated strong.
- BoardAdc__scanTask (0x420284dc): continuous ADC1 sampling configures four channels 1..4, 19200 samples/s and 256-byte read frames. GPIO15 and GPIO16 are driven from a two-bit mux state 0..3. ADC channel c in [1,4] maps to logical index 4*muxState+(c-1), giving sixteen slots, each averaged after 16 samples by a right shift of four and passed into PotSmoother__step. Hardware assignments to named knobs remain unverified. Rated strong for code-level behavior.
- NoteProcessor__setParam (0x42013f40): UI/engine descriptors identify ids 10..13 as SeqDir, SeqReset, SeqScale, SeqRoot, and ids 17..19 as ChordType, ChordScaledType and ScaleType. The 20-byte step record has directly written byte field at +0x789+20*step (subfield 0) and float field at +0x78c+20*step (subfield 1). Only ids 23/24 write the extra first-step fields +0x794/+0x798 in this routine; other subfields 2/3 are forwarded to the engine without a local write here. Unobserved record fields are not assigned speculative names. Rated strong for this routine, not a claim that every 20-byte field is understood globally.
- WavetableCache__init (0x4201682c): SALT comparisons at 0x4201693c and 0x420169bc choose table 88 for scratch rows greater than 60, table 0 otherwise. Two 8192-byte 64x128 signed scratch buffers are built identically from 64-byte waves, using wave byte minus 128 for samples 0..63 and reverse-order 127 minus wave byte for samples 64..127. Header word 100 and the users of these buffers remain unconfirmed external questions, not undocumented branches in this initializer. Rated strong.
- Sound__migrateValue (0x420294f0): completed the older-version conditional network, including modulo-84 per-part slots, Source and Target shifts, envelope-time maps, LFO square roots, and filter-type conversions with the distinct version <=6 and version 7 cases. The two bit masks 0x249249 and 0x2d600b00 select the claimed sets of per-part slot indices. Nonmatching paths and version >=8 return behavior checked. Rated strong.
- ReverbCore__init (0x4202ad50): fully checked the straight-line stores: packed ring indices (0,16), eight all-pass records, two modulated delay records, two diffuser records, two delayed-sum records, two seven-tap output lists, base coefficients and zeroed state fields before ReverbCore__updateTapGains. The design intent behind literal delay lengths and physical response cannot be proven by this initializer and is not asserted. Rated strong.

Two functions remain medium after this round:
- Voice__trigger (0x4037b61c): verified independent q-scaled random offsets and their destinations, envelope/retrigger state flags and mode-dependent coefficient setup. Still lacks complete mode-by-mode resonance-weight interpretation, bit-accurate approximation/reciprocal refinements and exhaustive trigger/retrigger branch semantics. No speculative mathematical interpretation promoted to a verified description.
- Voice__render (0x4037bfb0): verified oscillator wave decoding, phase indexing, LP/HP four-stage structure and mode-specific taps, stereo pan law and Drive cutoff, dry-add and gain paths in Rounds 91/92 and checked them again here. Full mode-dependent nonlinear saturation/feedback normalization and all control-tick branches still lack a complete proof. This is a code-coverage/interpretation limitation, not a missing function map.

Corrections to earlier notes: factory menu navigation is an encoder-0 event, not a button event; SALT/SALTU are valid comparison opcodes rather than undecodable instructions; the ADC task exposes logical mux slots, not named physical potentiometer wires. The exact meaning of the WavetableCache header and scratch-buffer consumers is not established by initialization alone. No firmware change was performed; the user-deferred queue-producer topic is still deferred. Independent unconfirmed topics listed in HANDOFF.md remain open.

Recheck status after this group: 0 rows still rated notes, 760 verified (strong), 2 medium.


## Round 99 - Voice trigger and render final DSP recheck

Rows: Voice__trigger `0x4037b61c` and Voice__render `0x4037bfb0`; row file `cls/map177.txt`. All conclusions are from static disassembly of the original v1.02 ELF, with no firmware modifications or runtime evidence. Full address-level derivations are in `protein/analysis/VOICE_DSP_MATH.md`.

- `Voice__trigger`: completed the mode-dependent LP and HP resonance-feedback coefficient arithmetic by tracing two cutoff stage factors through the bit-seeded approximate reciprocal and the four subsequent feedback weights. For `B(x)` defined in the math document, LP stages have `A=ga*B(1+ga)`, `D=gb*B(1+gb)`, `P=A*D`, `Q=2*r`, and their four feedback weights at voice+0x3e4 offsets +68/+72/+76/+80 respectively are `Q*D*P*B(1+ga)`, `Q*P*B(1+gb)`, `Q*D*B(1+ga)` and `Q*B(1+gb)`. HP stages have `A=B(1+ga)`, `D=B(1+gb)`, `P=A*D`, `Q=1.2*r` and weights at voice+0x400 offsets +132/+136/+140/+144 respectively `Q*P*P`, `Q*D*P`, `Q*P` and `Q*D`. LP12 scales resonance by 0.8, HP12 by 0.85. The associated pre-normalizers and post-gain stores are verified by their render consumers.
- `Voice__trigger` restart correction: `0x4037b8dd..0x4037b91a` restarts envelopes when `(int)(partFloatAt+0xe4)==1 OR (a6 low byte !=0)`, and otherwise retains stages 2..4 or sets stage 2. The a6-byte independently resets oscillator phases/marks both waveform buffers dirty at `0x4037b998..0x4037b9ae`; it is forwarded by VoiceSlot__start `0x42014b18`. The earlier claim that a glide flag simply suppresses envelope restart is not supported and has been withdrawn.
- `Voice__render`: soft saturation is `S(z)=z*B(abs(z)+bias)` where the float32 approximation is `t=float_of_bits(((0xbe6eb3be - floatbits(x))&0xffffffff)>>1)`, `B(x)=t*t*(2-x*t*t)`. This is not an exact reciprocal or division. Low-pass subtracts four weighted state feedback terms; high-pass adds them. Both update two alternating one-pole coefficients through four stages, selecting the second-stage result in 12 dB modes and fourth in 24 dB modes. Mode-specific coefficient stores from the trigger are read at render time or refreshed on control changes. Dry, Drive and stereo pan paths were cross-checked against Rounds 91/92.
- Four `excw` placeholders in the render wave-expansion loops at `0x4037d038`, `0x4037d160`, `0x4037d307`, `0x4037d430` all contain bytes `e0d872` and compare wave row against 60. They select table 88 for rows 61..63, preserving the original wave table for rows 0..60. No additional unresolved undecoded instruction remains in these two functions. The quarter-tick LFO update gates, q/eighth-tick envelope gates, modulation clamps, glide and phase update, gain ramps, noise LCG, filter mode dispatch and final 16-sample stereo gain/output branches are now covered in the revised descriptions and math document.

Corrections: previous 'exact reciprocal' shorthand is incorrect because the float-bit approximation has its own numerical response; previous 'glide suppresses envelope restart' summary is incorrect. The unexplained runtime contents of the clocked LFO ratios, external flag writers and audible/physical behavior are not assigned meanings beyond local code evidence. The unrelated user-deferred queue producer question remains deferred. No build-phase work performed.

Both rows now meet strong confidence **for their stated, bounded static descriptions**; this is not a claim that all system-level open questions are solved.

Recheck status after this group: 0 rows still rated notes, 762 verified (strong), 0 medium.


## Round 100 - system-level open-item audit

Static ELF only; no firmware changes, emulation or hardware. New detail file: `analysis/SYSTEM_OPEN_AUDIT.md`. Updated row descriptions: `cls/map178.txt` (Voice__init, VoiceBank__ctor, VoiceManager__ctor, Voice__render).

- Voice__init writes **18** clocked-LFO ratios at 0x3fcbb478..0x3fcbb4bf, using the 18 double divisors at 0x3c0702b0..0x3c07033f and constant 0.0005972586805000901. Earlier descriptions suggesting **19 initialized** ratios were imprecise. Voice__render nevertheless clamps the two ratio index paths to 0..18; entry 18 reads 0x3fcbb4c0, the first element of a separate interpolation table. Whether that index is reachable from supported user parameters is unconfirmed.
- Rechecked the part+0x391 steal-selection byte. VoiceManager__noteOn reads it. VoiceManager__ctor and VoiceManager__reset do **not** explicitly initialize that byte; the earlier preliminary claim that reset initializes it is corrected. A writer remains unconfirmed.
- VoiceBank__ctor initializes seven mapped ramp reload lengths: five 30-tick records at voice+0x36c/+0x37c/+0x38c/+0x39c/+0x3ac; a 100-tick record at +0x4b0; a 30-tick record at +0x4d0. These are exact packed halfword stores in the constructor. Other ramp interpretations remain outside this evidence.
- WavetableCache__init sets header s16 100 and zeroes 24 bytes at cache+0x56dea. Their *meaning* remains open; the two 8192-byte scratch tables have no verified external consumer. The DriveType DROM table needs no writer.
- Outstanding uncertainties concerning glide source and polarity, user wavetable scratch/cache consumers, sound-record fields, effects external controls, version discrepancy, absent storage partitions, and physical board wiring are maintained explicitly in `SYSTEM_OPEN_AUDIT.md`. The user-deferred queue-producer question was not pursued.

Recheck status after this group: 0 rows still rated notes, 762 verified (strong), 0 medium.


## Round 101 - LFO index reachability and glide-state semantics

Static ELF/parameter table only, no firmware/hardware changes. Detailed address-level evidence in `analysis/LFO_GLIDE_AUDIT.md`. Updated descriptions `cls/map179.txt`.

- The two LFO ratio parameters have 18 choices, indices 0..17, with normal pot, encoder and button paths confined to that range. Renderer indexing uses trunc(value + 0.05) clamped to 0..18, so a raw value >=17.95 could load from the first interpolator coefficient at 0x3fcbb4c0. The raw engine setter and stored-sound loading/push pipeline do not step-clamp imported ratio values, establishing a static route for malformed values. No occurrence or audible impact is claimed.
- GlideRate <0.01 bypasses pitch glide; at >=0.01, GlideMode 0 permits glide unconditionally, and nonzero GlideMode permits it only when a different note is also held. The trigger sets current pitch to previous note and step to abs(note-prev)/(3*44100*rate^3)*128 when permitted; the renderer approaches the new note with bounded steps.
- The VoiceSlot__start / Voice__trigger flag is a retrigger/start mode, not the GlideMode sound parameter: 0 snaps three per-note controller ramps to their targets (used on mono voice reuse), 1 ramps them using 10-block reloads (used on fresh starts), independently affecting envelope/oscillator restart. Arrays are Poly Aftertouch, Note PB and CC74/MPE Y.
- Important correction: VoiceManager__reset writes u16 0x0100 to +0x390; the high byte at +0x391 therefore becomes 1 by default, resolving its initializer, contrary to Round 100. A later override remains unconfirmed.
- No queue-producer investigation and no firmware modification. Other system-level unresolved topics remain.

Recheck status after this group: 0 rows still rated notes, 762 verified (strong), 0 medium.


## Round 102 - Wavetable bank consumers and cache-header audit

Only ELF static analysis; new address-level report `analysis/WAVETABLE_CONSUMER_AUDIT.md`, updated rows in `cls/map180.txt`. No firmware or hardware changes; deferred engine-event queue producer untouched.

- **Correction:** The two 8192-byte tables at shared VoiceEngine+28 and +0x2024 are NOT unconsumed. `Voice__init` (0x42015dd1..0x42015dde) saves pointers to the two descriptors at shared+20 and shared+0x201c into voice+56 and voice+204. Each descriptor has cache pointer +0, last-selected table number +4, and a 64x128 signed-byte wave bank +8. `Voice__render` dereferences these pointers to load selected 128-byte rows at 0x4037d08c..0x4037d096 / 0x4037d1b5..0x4037d1bf and alternate paths 0x4037d35c..0x4037d366 / 0x4037d485..0x4037d48f. A change in selected table triggers a full 64-row bank rebuild from the main cache and updates descriptor+4; matching table avoids rebuilding.
- `WaveSection__draw` uses the global cache pointer at 0x3fcbdcc0 directly, not these predecoded banks.
- Header u16 100 at cache+0 and the 24-byte zeroed region at cache+0x56dea have no verified semantic readers. The region lies between the end of the 24x61 u16 user map at +0x56dea and the start of the 122x64 byte user pool at +0x56e02. Their exact purpose remains **unconfirmed**; do not invent field names.
- Other open questions remain. No hardware, emulator, or firmware build; no deferred queue-producer investigation.

Recheck status after this group: 0 rows still rated notes, 762 verified (strong), 0 medium.

## Round 103 - Eight requested outstanding system topics consolidated

Original v1.02 ELF and UF2, static code only. Detailed verified and unconfirmed results, with address-level evidence, are in `analysis/ROUND103_EIGHT_AREA_AUDIT.md`. Revised row descriptions in `cls/map181.txt` (9 rows). The firmware image and owner's deferred queue-producer question were not changed or pursued.

1. Sound/preset: the previously named `SoundLibrary__isUsed` at 0x4204f934 returns the **category** header byte +24, not the independent occupancy byte returned by `LibraryIndex__isUsed` 0x4204f900. Header byte +25 is the format number, and `Sound__resetEditState`'s little-endian `0x0800` yields category 0 / format 8. The 18 category strings were decoded. `FrontPanel__ctor` initializes the Chord/Pan/LFO/FX local state; it also clears Master raw volume and the 16-bit Master overlay counter in the same store, correcting an older unresolved reset item. Stored user presets remain absent from the image.
2. Sequencer/chord: verified the 20-byte step record (type, pitch, velocity, gate, two values), defaults, that normal editing writes pitch and velocity, and that additional parameter writes affect only step 0. Chord note lists are generated on note-on, with independent scale-mask parameter refresh and 17 absolute-chord/six scale-degree lookup definitions. No normal rest/tie step-type writer or extra modulation consumer established.
3. Wavetable: header `100` and the 24 bytes between the user-map and wave pool have clear initializer stores but no verified semantic readers. Their meanings remain unconfirmed; the two 8 KB playback banks were already resolved in Round 102.
4. LFO ratio: normal UI 0..17 versus renderer 0..18; a malformed raw float >=17.95 can select the first float of the neighboring interpolation table. Static import/engine route exists; no hardware/audible occurrence established.
5. Effects: resolved Delay and Tremolo timing source to TempoClock+24 **samples per beat**. Tremolo secondary sync uses ArpSeq runner-active and RhythmClock+40 last-step beat position relative to TempoClock+48 current beat position. Tremolo sync selector +0x74, Delay offset +0x40 and other optional effect writers remain unconfirmed.
6. Voice: byte manager+0x391 defaults to 1 via reset's overlapping u16 store; no later override verified. Six part controller ramp sources, three per-note ramp sources and seven DSP reload defaults are distinguished.
7. Board: exact 4-channel x 4-state ADC logical map, pot id->slot permutation, active-low serial input bits, LED serial output masks, and GPIO bit-bang roles confirmed. External physical pot wiring, LED electrical polarity and button positions are not determinable from the image.
8. Version: original image and ELF identify v1.02 b2-232 git dirty, whereas both hard-coded info screen and SysEx report 1.03 b3. Root cause remains unconfirmed.

No new functions identified and no firm system-level conclusions invented where the image lacks a reader, writer, hardware or history. Recheck status after this group: 0 rows still rated notes, 762 verified (strong), 0 medium.

## Round 104 - Final investigation of the owner's nine open-item notes

Original `protein_v102.elf` and `PROTEIN.UF2` only; static evidence, no hardware, emulator, patch, build or deferred queue-producer research. Detailed trace, precise offsets and limits: `analysis/ROUND104_OPEN_QUESTIONS_AUDIT.md`.

1. **Correction to Rounds 102-103:** Cache `+0` is not a field without readers. `WavetableCache__buildSynthTables` table 69 `HeavyFuz` writes and reads it as the 16-bit LFSR state: `s16i` at `0x4202c3c2`, `0x4202c3f9`, `l16ui` at `0x4202c41a`. Initialization writes u16 100, but this builder starts from local seed 200 and overwrites that value before the first identified feedback-state read. The location's purpose is now established; the purpose of literal 100 independently is not. The separately zeroed 24 bytes at cache+`0x56dea` still have no verified semantic consumer.
2. Voice ramp consumer audit names constructor's seven reload defaults: +0x36c osc1 wave position 30; +0x37c osc2 wave position 30; +0x38c osc1 level 30; +0x39c osc2 level 30; +0x3ac noise level 30; +0x4b0 filter cutoff 100; +0x4d0 volume 30. `Voice__trigger` sets pan +0x4e0 directly from part+212 and volume +0x4d0 from squared part+220, with counters zeroed. These sources are separate from the start flag's three controller ramps and true portamento.
3. VoiceManager+0x391 default 1 via the 16-bit reset write `0x0100` is confirmed, but a later indirect override is not. The five immutable DriveType floats are 0.2, unchanged. No additional user-facing writer of effects' hidden sync/modulation fields was verified; Reverb+0x418 does have a confirmed `Reverb__process` reader at `0x420134ec` and is not dead storage.
4. Both 8192-byte decoded wavetable banks are confirmed live Voice__render inputs (Round 102). The sequencer's two per-step floats are published at +0x308/+0x30c, but their user-editable destinations or later musical consumers remain unconfirmed. Step rest/tie handling and chord rebuilding are implemented; not all fields have confirmed normal UI setters.
5. LFO UI indices 0..17 versus renderer 0..18 remain a static boundary mismatch if imported floats are not normalized. No observed audio fault. Logical board mapping is established but not electrical physical wiring; independent hard-coded 1.03 b3 display/SysEx values conflict with app 1.02; the binary cannot establish historical cause. Original UF2 does not include saved flash partitions.

`cls/map182.txt` updates five strong-rated function descriptions with corrected cache and ramp meanings and properly bounded sequencer consumer language. No change in class, status or confidence beyond these descriptions. No new function identified.

Recheck status after this group: 0 rows still rated notes, 762 verified (strong), 0 medium.

## Round 105 - Deeper cache-header, 24-byte region and indirect writer audit

Detailed evidence and exact addresses: `analysis/ROUND105_CACHE_WRITER_AUDIT.md`. Original ELF and UF2 only, no hardware or firmware changes.

1. `WavetableCache__init` writes u16 100 to cache+0 only on the first allocation; it immediately builds all 89 tables. The HeavyFuz builder starts from a local seed 200 and stores a derived u16 to cache+0 before its first identified read. Therefore 100 does not seed HeavyFuz's generated waves. Historical purpose of the initial 100 remains unproven.
2. The 24-byte region `[cache+0x56dea, cache+0x56e02)` is independently cleared as 12 u16 values. The 24x61-u16 user map ends immediately before the gap, and the 122x64-byte user-wave pool begins after it. The single ELF literal 0x56dea is used by initializer only. No identified valid user-table read reaches the gap; alias/computed readers cannot be excluded. Semantic role remains unknown.
3. Checked the public effect setter and constructors: Tremolo+0x74, Delay+0x40 and Reverb+0x418 are initialized to 0, 0 and 0.5 respectively but are not assigned by the exposed FX IDs 0..74. Reverb's default derives a core coefficient pair (1,1) when its update branch runs. Optional modulation-source pointers likewise have no identified public setter. No global absence claim for indirect writers is made.
4. Normal `VoiceManager__noteOn` writes **only byte +0x390**, preserving the adjacent +0x391 steal-mode byte set to 1 by reset's 16-bit initialization. A later override remains unconfirmed, but the round-robin allocator itself does not change the flag.

Recheck status after this group: 0 rows still rated notes, 762 verified (strong), 0 medium.


---

## Historical appendix - original Round 105 handoff (supplied 2026-11-06)

This is a **preserved snapshot** of the stock-code research phase. Any assertion below that the project has not entered firmware modifications, that emulator evidence is never used, or that the future features are unbuilt is an **historical project rule**, superseded by the later experimentally patched working baseline in `Protein_Handoff.md`. Stock-firmware analysis results and R105 open-item bounds remain relevant.

# Waldorf Protein v1.02 reverse engineering - handoff

Written Tue 2026-11-03 for continuing the project with another assistant. Updated after Round 105 (deeper cache and hidden-state writer audit). Everything below is from static reading of the firmware only. Nothing has been tested on hardware.

## 1. Goal and hard rules

Owner: Rico (hobbyist, live dawless techno). No Protein hardware. Will not dump the flash. So all results must come from the firmware image alone, and emulator output is not accepted as evidence.

Final aim (IMPROVEMENTS.md), to be built LATER:
1. Arp/sequencer latch that works with the arp.
2. Glide/Wave/Filter/Amp knobs control the ADSR of a selectable envelope.
3. Custom wavetables via a Windows tool that rebuilds the firmware.
4. Envelope times shown in ms/seconds.

Current phase (set by the owner): "We are not going to make modifications until our understanding of the code is 100% complete." So NO build-phase work. The RATING RECHECK of class P notes rows is complete through Round 84. All identified Protein functions are now strong-rated; remaining work concerns explicitly unconfirmed whole-system behavior, not building.

Owner's standing rules for replies:
- No emojis. No em-dashes (use a plain hyphen). No curly quotes.
- Short, concise answers. One paragraph plus one bullet block at most. One idea and one question per response.
- Anything to be copy-pasted goes in a code block.
- Flag anything unconfirmed. Do not guess or invent. Do not describe how the work was done.
- Ask a clarifying question if unsure.
- End each group report with ONE question about the next group.
- Do not raise the sound-format version bump (8 to 9, migration function 0x420294f0 area) until the build phase.
- Deferred by the owner (keep deferred, do not chase unless asked): the "queue producer" question (who sends the engine events of types 11 to 13 and what feeds the event queue 0x3fca3960).
- When the owner is away, work unattended: pick the most reasonable reading, say which, carry on.

## 2. Files (all in the package zip)

- PROTEIN.UF2 - original manufacturer firmware file provided by the owner; included here for archival comparison with the ELF.
- protein/protein_v102.elf - the image as an ELF (ESP32-S3, ESP-IDF 5.5.1, "Chutney_Firmware" v1.02_b2-232-gdda56c3c). Open in Ghidra, IDA or radare2. Segments: DROM 0x3c060020, DRAM 0x3fc9da00, IRAM 0x40378000, IROM 0x42000020 (340,856 bytes), IRAM 0x4037fa84, RTC 0x50000000. Note: stock Ghidra lacks the Xtensa LX7 SALT/SALTU instructions (patch in the original project, tools_patches/).
- protein/analysis/function_map.csv - THE MAIN RESULT. Columns: addr,size,class,class_evidence,status,name,library_module,confidence,description. class P = Protein code (762 rows), L = library (1638), DATA = verified literal/data (2). There are no U or DATA? entries. Only class P rows are included in the Protein rating counts.
- protein/NOTES.md - the full log, Rounds 1 to 105. Rounds 73 to 84 recheck notes-rated rows; Rounds 86 to 99 revisit medium-rated rows; Round 100 audits system-level uncertainties; Round 101 resolves LFO UI indexing and glide-state semantics; Round 102 identifies the shared decoded-wavetable bank consumers. Round 103 investigates eight owner-requested system topics; Round 104 follows up nine outstanding notes, corrects the wavetable cache+0 semantic reader claim and identifies seven voice DSP ramp destinations. Round 105 narrows the initial-100 question, confirms the 24-byte gap has no verified direct semantic access, and traces hidden-state setter boundaries; see `analysis/ROUND105_CACHE_WRITER_AUDIT.md`. Read the last rounds first.
- protein/KNOBS.md, PARAMS.md, BUTTONS.md, IMPROVEMENTS.md, analysis/ui_params.txt (UI parameter index; engine id = UI index + 0x95), envelope_times.csv.
- cls/ - helper scripts and the per-group row files map100.txt to map181.txt (format below).

## 3. Method used for each group

1. Take the group's rows (class P, confidence notes). Dump each function's disassembly, read it fully, follow callees only as far as needed.
2. Write one line per function in cls/mapNNN.txt: `0xADDR|Name|description|confidence` (name may be empty to keep the old one; description must not contain the | character; no em-dashes or curly quotes).
3. Apply with `python3 cls/apply.py mapNNN.txt` (edits function_map.csv in place, sets status "described (Round 48)"). Then `python3 cls/stats.py` for counts.
4. Append a "Round N" section to NOTES.md: findings verified in code, corrections to older notes, rows rated medium and why, still-unconfirmed items, last line "Recheck status after this group: X rows still rated notes, Y verified (strong), Z medium."
5. Rating: strong = read in full and every claim checked against the code. medium = read in full but some detail (formula, field meaning) not re-derived; say exactly what. notes = not yet rechecked.
6. Send a short report ending with ONE question about the next group.

Tools: cls/rd.py requires the original project paths. Round 83 instead used cls/rd_cf.py and cls/disasm_group.py with the user-supplied Espressif xtensa-esp32s3-elf toolchain (also not included in the zip). They expect toolchain/xtensa-esp-elf/xtensa-esp-elf/bin/objdump relative to project root. The CLI objdump sometimes shows LX7 SALT/SALTU as excw; their raw bytes were separately checked before upgrading the relevant rows in Rounds 98 and 99. cls/xref.py finds words equal to an address. cls/build.py has rd32/rdstr helpers over the ELF. A literal-pool scan (decode L32R operands over the IROM bytes) was used to find every reader of a RAM offset; this is reliable but reports a few misaligned false hits.

## 4. Status

Class P rows: 762. Rechecked groups (all rows now strong or medium):
1 storage and persistence (R73), 2 boot and hardware init (R74), 3 MIDI small and mid (R75), 4 MIDI core (R76), 5 effects (R77), 6 arpeggiator and sequencer (R78), 7 parameter and preset (R79), 8 front panel and UI (R80), 9 engine and voice (R81, 43 rows), 10 wavetable cache (R82, 4 rows), 11 board, factory tests, display and ADC (R83, 19 rows), 12 formatter and boot splash (R84, 2 rows). Round 85 separately classified 19 non-P candidates: 17 L and 2 DATA. Rounds 86-90 revisited 23 medium-rated P rows in five topic groups.

Counts now: 762 strong, 0 medium, 0 still rated notes (Round 105; no rating changes). Rounds 86-90 revisited all 23 medium-rated rows, upgrading four to strong; Round 94 upgraded two ReverbCore rows; Round 95 upgraded Reverb__process. Round 91 resolved random-field destinations, 64-to-128 waveform decoding and pan gain laws. Round 92 verified LP/HP stage recursions, tap selections and drive amount/blend/gain paths. Both voice rows were subsequently resolved in Round 99 (details in VOICE_DSP_MATH.md). Round 93 corrected effects exponential cutoff coefficients, derived a Color piecewise reverb wrapper law and the Tremolo basic sync-rate calculation, without rating upgrades. Round 94 resolved ReverbCore diffuse arithmetic and process block paths, upgrading those two rows to strong; its initialiser was upgraded later in Round 98. Round 95 traced the Reverb wrapper against those core paths and upgraded it to strong. Round 96 upgraded Chorus__process and Delay__process after confirming compensation flag, stereo routing, interpolation and output laws. Their external state/source questions remain open but are not unresolved function-level arithmetic. Round 97 upgraded Phaser__process and Tremolo__process by verifying exact Phaser model coefficients, recurrence, and Tremolo double-clock phase correction. Their remaining runtime source identities and physical responses are not asserted. Round 98 resolved the remaining ten nonvoice medium rows, including SALT/SALTU comparisons, ADC mux slot derivation, six factory-test callbacks and menu encoder mapping. The two voice rows were upgraded in Round 99 after resolving their nonlinear arithmetic and local control paths.

All 762 identified Protein functions are now strong-rated (descriptions only); no Protein rows remain medium or notes. Round 99 resolved Voice__trigger 0x4037b61c (mode-dependent LP/HP resonance-feedback weights, approximation and restart conditions) and Voice__render 0x4037bfb0 (LP/HP soft saturation, control gates and the four row-selection SALT opcodes). See NOTES Round 99, cls/map177.txt, and protein/analysis/VOICE_DSP_MATH.md. External system-level open items listed in section 5 remain open; do not conflate strong function descriptions with complete system understanding.

## 5. What is still required

A. No class P rows remain rated notes. Round 84 upgraded the formatter and boot splash to strong. Round 86-90 upgraded the note-off handler, text-entry encoder, ADC smoother initializer and reverb tap-gain updater to strong. Round 85 classified all 19 previously unclassified entry candidates: 17 platform/library routines and 2 literal/data spans; no new Protein-specific code was identified. See cls/map163.txt, NOTES Round 85 and protein/analysis/MAPPING_AUDIT.md. Module-specific details of several new library classifications remain medium confidence.
B. Round 98 rechecked all 12 remaining medium functions, upgrading 10 to strong: NoteProcessor__setParam, WavetableCache__init, TempoClock__advance, FactoryTest__midiTest, FactoryTest__runMenu, Board__scanInputs, Led__encodeOutput, BoardAdc__scanTask, Sound__migrateValue and ReverbCore__init. Round 99 upgraded the two remaining voice DSP functions after recovering their mode-specific nonlinear coefficient math and branch paths. Remaining work concerns system-level open questions, as listed below; a strong rating does not imply hardware or complete whole-firmware behavior is verified.

C. Unconfirmed items still open (all flagged in NOTES):
- Voice steal-mode byte at part+0x391: VoiceManager__reset DOES initialize it to 1 via an overlapping u16 store of 0x0100 at part+0x390 (Round 101 corrects Round 100). Possible later overrides remain unidentified.
- The ratio table contains 18 initialized floats at 0x3fcbb478..0x3fcbb4bf, matching the LFO ratio descriptors (18 UI choices, indices 0..17). Voice__render nonetheless clamps to 18 inclusive, which reads the first coefficient of the adjacent sinc/Kaiser table at 0x3fcbb4c0. UI writers keep 0..17; an unvalidated imported value >=17.95 has a static path through sound loading and engine parameter updates (Round 101). No observed runtime bug is asserted.
- Glide clarified Round 101: GlideRate and GlideMode control true pitch portamento (rate >=0.01, mode 0 unconditional, other mode requires a different held note). The 0/1 VoiceSlot__start flag is a separate retrigger/start mode for controller ramps, envelopes and oscillator phases: 0 direct, 1 ten-block ramp for Poly Aftertouch, Note Pitch Bend and CC74/MPE Y. No remaining basic glide flag-polarity ambiguity; intended UI labels/acoustic response not asserted.
- DriveType 5-entry table at DROM 0x3c07002c is immutable and consists of five 0.2 floats (verified Round 92). VoiceBank__ctor sets five ramps to 30 ticks (osc1 and osc2 wavetable position, osc1 and osc2 levels, noise), one to 100 ticks (filter cutoff), and one more to 30 ticks (voice volume), verified by their Voice__render consumers (Round 104). Later reload overrides may remain open; pan has a separate direct init record.
- **Resolved in Round 102:** The two 8192-byte banks at shared VoiceEngine+28 and +0x2024 ARE read by Voice__render. Voice__init installs descriptor pointers (shared+20 and +0x201c) at voice+56 and +204; a descriptor holds main-cache pointer, selected table id and 64x128 predecoded wave bytes. The renderer rebuilds a bank from the main cache when the selected wavetable table changes, then copies a 128-byte row to voice-local oscillator samples. See analysis/WAVETABLE_CONSUMER_AUDIT.md.
- **Resolved in Round 104:** Cache+0 is a 16-bit LFSR feedback state in HeavyFuz table 69. WavetableCache__buildSynthTables writes and reads cache+0 at 0x4202c3c2, 0x4202c3f9 and 0x4202c41a. The cache initializer literal100 is overwritten by a local-seed200 path before this identified reader; why initializer wrote 100 is unconfirmed. The separate 24 bytes at cache+0x56dea are initialized as 12 u16 zeros; the unique 0x56dea literal is used only by the initializer. No verified semantic reader. Round 105 demonstrates that the earlier u16 100 cannot seed the normal HeavyFuz builder because local 200 is used and a derived state is stored first.
- **Resolved in Round 103:** Chord/Pan/LFO/FX section UI initial states, library header byte +24 category (independent of occupancy), Master overlay counter reset, 20-byte sequencer step structure, and chord rebuild on note-on. The normal user editing paths for step rest/tie types and extra per-step modulation destinations remain unconfirmed. Parameter names for ids 10-13 and 17-19 are verified from descriptors.
- **Version mismatch (Round 103):** Both the info page and SysEx version reply hard-code 1.03 b3, independently from the image/ELF app description v1.02. Why these versions differ cannot be proved from the binary.
- Storage: bootloader, partition table and the NVS/data partitions (saved sounds and globals) are not in the UF2, so the save format of user data cannot be checked.
D. Effect external open checks: Tremolo +0x74 tempo-sync selector writer and optional effect parameter-source identities remain unconfirmed. Clock units/source and nested clock at +0x88 are resolved by Round 103: TempoClock samples-per-beat and ArpSeq/RhythmClock last-step beat position. Chorus +0x200 external purpose and Delay +0x40 writer remain unconfirmed.
E. Board-specific open checks: logical mux-to-pot id mapping is now established by Round 103, but physical wiring and LED active-high/low hardware polarity remain unverified without the PCB/hardware. FIFO and timer comparisons, logical LED blink condition, factory MIDI UART ring framing and encoder-0 factory-menu navigation are now proved in Round 98; physical button legend/position meanings remain open. Pot smoother coefficient formula was verified in Round 89.
F. Deferred by the owner: queue-producer question (see section 1).
G. After the owner declares understanding complete: design and build phase for the four improvements (not started, nothing built or tested).

Round 100: `protein/analysis/SYSTEM_OPEN_AUDIT.md` records the initial open-item audit, with its claim that +0x391 lacked a reset writer superseded by Round 101. Round 101: `protein/analysis/LFO_GLIDE_AUDIT.md` proves normal UI LFO bounds, the raw-value out-of-range path, pitch-glide conditions, per-note controller ramp semantics, and the overlapping u16 voice-steal initializer. `cls/map179.txt` contains the revised strong-rated descriptions. Round 102: `protein/analysis/WAVETABLE_CONSUMER_AUDIT.md` identifies the two previously unidentified predecoded-bank consumers and records unresolved cache metadata. `cls/map180.txt` updates six strong-rated descriptions. Round 103: `protein/analysis/ROUND103_EIGHT_AREA_AUDIT.md` addresses eight owner-requested areas. Round 104: `protein/analysis/ROUND104_OPEN_QUESTIONS_AUDIT.md` corrects the cache+0 read/write interpretation and specifies the remaining limits for all nine last-requested topics. `cls/map182.txt` updates five descriptions. Round 105: `protein/analysis/ROUND105_CACHE_WRITER_AUDIT.md` checks the first-build state overwrite, the independent 24-byte gap, normal public FX parameter assignments, and the byte-width of round-robin updates; `cls/map183.txt` expands seven descriptions. Remaining uncertainties are explicitly bounded, rather than treated as failures of mapping. No firmware changes or queue-producer investigation.

## 6. Key verified facts (for orientation)

Engine
- Engine task (0x4200ccc8) and audio output task (0x4200ca8c) form a double-buffered pipeline of 16-sample blocks at 44.1 kHz. Engine object 0x3fca3974 holds six 16-float bus buffers. Event queue 0x3fca3960 (24-byte events). Output stage multiplies bus 0 by 2^31 with saturating conversion.
- Engine event types (dispatcher 0x4200d7f4): 0 note on, 1 note off, 2 pitch bend, 3 CC, 4 channel pressure, 5 poly aftertouch, 6 set param, 7/8/9 further param slots. Types 10 to 13 act in the sender (Engine__sendEngineEvent 0x4200d52c): 10 master volume (value/511), 11 LFO number, 12 layer for LED mirror, 13 Fx__resetAll. Type 6 also calls Fx__setParam immediately.
- Master volume gain = level squared, with a ramp.
- Control tick is 49 samples (plus random -32 to +30 when q > 0.25).
- Voice manager: 8 voice slots per part (state 0 free, 1 playing, 2 released, 3 fading). Victim order: free, oldest released, then oldest playing (oldest fading if byte +0x391 set), else highest note. 32 voice objects at 0x3fcaaf78, stride 0x828, vtable 0x3c070288 (+0x10 trigger, +0x14 release, +0x18 finished, +0x1c fast fade, +0x20 setParam, +0x24 render).
- Envelopes: stage bytes 1 attack, 2 decay, 3 sustain, 4 hold, 5 release, 6 finished; coefficient table 0x3fccde98 indexed by trunc(value x 127). LFO shapes 0 sine, 1 triangle, 2 square, 3 falling ramp, 4 rising ramp, 5 noise, 6 sample and hold; skew fixed 0.5.
- Wavetable step mapping: UI wavetable numbers above 31 select internal n + 24, so the 24 User tables (internal 32 to 55) cannot be reached from the knob.

Wavetable cache (relevant to improvement 3)
- Global pointer 0x3fcbdcc0, 0x58c83 bytes in PSRAM. Layout: s16 +0 initially = 100 (later used as HeavyFuz table 69 LFSR state); 89 tables x 61 waves x 64 bytes from +4 (table t at +4 + t x 3904); flags 89 x 61 bytes at +0x54d44; user table map 24 x 61 x u16 at +0x5627a (tables 32 to 55); user wave pool 122 x 64 bytes at +0x56e02.
- Wave numbers: 0 to 245 ROM waves, 246 to 367 user pool, 368 to 421 ROM waves 246 to 299, 0xffff = empty (filled by averaging neighbours), anything else = ROM wave 101. ROM waves 0x3c079740 (300 x 64 bytes, unsigned 8-bit centred on 128). ROM table definitions 0x3c07e240 (40 lists x 61 x u16: tables 0 to 27 and 76 to 87).
- Build order: user tables 32 to 55 not built (empty), 28 to 31 and 56 to 75 code-built, table 88 special, the rest from ROM definitions.
- No code in v1.02 writes the user table map or user wave pool. Names of tables are an 88-entry pointer array at 0x3fca05ac; entries 32 to 55 are "User   1" to "User  24".

Other
- Parameter descriptor table 0x3c071f54, 32 bytes per entry (108 parameters, see PARAMS.md).
- Current sound format version is 8. Any format changes are deferred to the later build phase.
- Format of rows, names and every group's detail: see NOTES.md Rounds 73 to 105.

---

# Custom-firmware supplement: compressor and panel changes (12 November 2026)

**Supersedes earlier statements that no firmware patches have been built.** The original decompilation Rounds 1-105 above remain untouched as a historical stock-firmware reference. This supplement describes **experimental modifications**, not newly discovered stock functions. See the companion `Handoff.md` for the current implementation state and `Functions_Map.csv` for the custom-patch address entries (class `PATCH_*`).

## Source and status

- Factory image: `PROTEIN(3).UF2`, factory SHA-256 `ffcd702affd22a8822506ebd47327f3535a268868761b42bf0adacb23b294532`.
- Original compressor-free custom baseline: `PROTEIN_Working.UF2`, SHA-256 `71d51b8a77f71dfa8b8a4da950be3de6ea20f70c9bcfb011e1da4bace4fc4849`. Contains accepted ADSR and four-note latch work.
- Last user-confirmed stereo-register correction: Stage 7, `PROTEIN_STAGE7_CORRECT_LEFT_REGISTER_EMULATOR_ONLY.UF2`, SHA-256 `b262a033440db5766618d6d82d205a1ef9668108a51335ebf8fee5bbd7607a9d`; artifacts greatly reduced, not eliminated.
- **Current latest generated build**: Stage 10, `PROTEIN_STAGE10_FLAVOUR_GAIN_MINUS8_THRESHOLD_MINUS30_EMULATOR_ONLY.UF2`, SHA-256 `5b6919d62a1966e38b491afbcffe5128bccab1d1b3caeed197dd6ededb695e53`; decoded application SHA-256 `e6b47c7e67c31fe53bceb1f45d96e93223294906bc5baae34fd46d5f9febc1da`.
- **Validation limits:** Stage 10 passed image integrity, precise changed-address allowlists, and a restricted instruction-level interpreter for nine Flavour positions. The user has **not yet reported an emulator test of Stage 10**, and **no build is hardware-verified**. All new UF2 files are **emulator-only** and should not be flashed onto the physical Protein.
- Working sources: stage-specific `build_stage*.py`, `static_validation.json`, `mapper_instruction_validation.json` and READMEs under the supplied `Protein_Stage*` / `Protein_Compressor_Stage*` folders. The independent third compressor is a custom addition and should not be attributed to factory firmware.

## Stage history and regressions

| Stage | Modification | Evidence / finding |
| --- | --- | --- |
| 1 | Create an independent compressor state and boot-time constructor/init trampoline. | Emulator boots. Original Stage 1 hook passed the wrong Xtensa CALL8 argument register; boot alone did not validate initialization. |
| 2 | Add compression to audible output using scratch/copyback. | Initial build silent; passthrough and discard-output diagnostics audible. Correcting initialization argument handling restored audio. User heard higher loudness, consistent with AutoGain but not a measured gain-reduction test. |
| 3 | Assign Flavour to independent compressor ratio; relocate compressor state. | Initial state `0x50000030` overlapped the emulator input/button mailbox at `0x50000040`; latch-off failed. Moving state to `0x50000100` resolved the observed latch regression. |
| 4 | Move compressor **after master EQ and before master limiter**. | User reported compressor worked, but high-pitched artifacts appeared only on the left output. Compressor-bypass diagnostic was clean; bypass also removed compressor by design. |
| 5 | Fix LFO Source/Shape and Target encoder presses; Init Sound PAN defaults. | User reported feature set working. Presses must enter ModMatrix without changing a parameter. Init Sound: RandPan = `Fix` / step 0; Pan Position = factory raw 0.5, displayed as center `0`; old presets preserved. |
| 6 | Pass the original left and right pointers rather than deriving left = right - 64. | Artifacts remained left-only. This build also had a displaced-instruction register error (corrected Stage 7). |
| 7 | Restore the correct left pointer before replaying original limiter instructions. | User reported artifacts greatly decreased, appearing sometimes at maximum Flavour. Compressor-output L/R-swap diagnostic still produced artifacts on the left. Root cause remains unproved; could involve limiter, final output, or emulator handling. |
| 8 | Flavour ratio + threshold + output-gain control; 200 ms release. | Static tests passed; threshold 0 to -25 dB, output gain 0 to -20 dB. User said output too quiet. |
| 9 | Output gain changed to 0 to -10 dB. | Static tests passed; user said still too quiet. |
| 10 | Output gain changed to 0 to -8 dB; threshold 0 to -30 dB. | Static tests passed, including instruction-level mapper tests. No recorded Stage 10 emulator audio or hardware test as of this update. |

## Present Stage 10 compressor design

Signal order: **Synth / voices -> original FX1 -> original FX2 -> master EQ -> independent Flavour compressor -> original master limiter -> output**. This placement is intentional for now, but may be reconsidered after physical-unit listening.

The new instance uses the original Waldorf compressor algorithm and an independent **0x440-byte** state object at `0x50000100..0x5000053f`, separate from the compressor shared by the original FX slots. FX1 and FX2 can select Compressor, but the Flavour compressor does **not** share their controls or DSP state. The original FX slots keep their stock behavior, including their existing shared-compressor limitation when both are set to Compressor.

With normalized Flavour value `x` in [0,1]:

- **Ratio:** `1 + 99*x^3` (1:1 at min to 100:1 at max); native nonlinear response.
- **Threshold:** `-30*x` dB (0 to -30 dB).
- **Output Gain:** `-8*x` dB (0 to -8 dB).
- **Input Gain:** 0 dB, fixed.
- **AutoGain:** On, fixed.
- **Attack and LookAhead:** separate compressor's initialized defaults.
- **Release:** fixed at 200 ms; the patch writes the stock-mapped second-stage release coefficient (`~0.9989562631`, observed in Stage 10 instruction test).
- **Flavour OLED overlay and preset parameter** (`flav`, UI 0x22f, engine id 20) are preserved. Turning it drives the new compressor via a custom audio helper.

| Flavour | Ratio | Threshold | Output gain |
| ---: | ---: | ---: | ---: |
| 0% | 1:1 | 0 dB | 0 dB |
| 25% | 2.55:1 | -7.5 dB | -2 dB |
| 50% | 13.375:1 | -15 dB | -4 dB |
| 75% | 42.765625:1 | -22.5 dB | -6 dB |
| 100% | 100:1 | -30 dB | -8 dB |

**Important: at Flavour 0%, this is not a true bypass.** The stock `Compressor__kernel` applies an AutoGain multiplier containing a 0.8 factor, approximately -1.94 dB, even at 1:1. Do not describe minimum Flavour as unchanged sound or unity signal. The compressor kernel's noted signed-only peak detector is another unresolved DSP limitation; see the stock map entry for `0x4202a008`.

## Relevant custom code, data and hooks

| Address / region | Stage 10 role |
| --- | --- |
| `0x4200f7bc` | Existing FX setup call redirected to compressor-initialization trampoline. |
| `0x42054800..0x4205483f` | Additional compressor constructor/init trampoline plus literals; preserves original FX initialization. |
| `0x50000100..0x5000053f` | Independent compressor state; **emulator-only address reservation**, not yet hardware-safe. |
| `0x4200f297` | Pre-limiter hook inside stock `Fx__process`, replaces a six-byte original-instruction window via wrapper. |
| `0x42054900..0x420549ff` | Stereo scratch-buffer compressor call/copyback path; reads Flavour and invokes stock `Compressor__process` `0x4200fb78`. |
| `0x42054a00..0x42054afb` | Stereo wrapper, including Stage 7 corrected `MOV a3,a2` before displaced left-buffer float loads. |
| `0x42054840..0x4205484f` | Fixed mapper float literals (0.5 baseline, 2/15 output-gain slope, 200ms release coefficient, 0.6 threshold slope). |
| `0x42054850..0x42054890` | 65-byte Stage 10 mapper helper; maps Flavour to threshold and output gain, writes release coefficient. |
| `0x420548f8` and `0x420548fc` | Pointers to Flavour engine value (`0x3fcaa844`) and independent compressor state. |
| `0x3c0732f4` | FrontPanel onEvent vtable slot redirected from original `0x4201e640` to custom trampoline `0x42054b00`. |
| `0x42054b00..0x42054b7f` | LFO button-press routing trampoline: Source/Shape press id 14 or Target press id 13 opens ModMatrix mode 2 via zero-delta synthetic encoder event; turns remain factory. |
| `0x3c072944` | RandPan default record overridden for Init Sound from Rand1 (step 1) to Fix (step 0). Pan Position remains centered. |
| `0x4200f685` | Restored factory post-limiter clipping call; no longer hosts the compressor. |

These are **modified-firmware patch sites**, not new stock decompilation discoveries. The complete original `Functions_Map.csv` 2,402 records are retained and the current patch entries are separately flagged as `PATCH_CODE`, `PATCH_SITE` and `PATCH_DATA`.

## Remaining tasks / experiments

1. **Emulator Stage 10 regression test:** audio, Flavour response, stereo output, four-note latch on/off, LFO encoder presses, Init Sound PAN Fix/0, original FX1/FX2, preset loading and parameter recall.
2. **Find left-only artifact source:** measure per-channel signals before/after compressor, limiter and emulator output (including possible pointer/register and FPU preservation problems). Stage 7 reduced the issue; channel-swap diagnostic did not relocate it. A suspected emulator fault is unconfirmed.
3. **Prepare a hardware-safe build and recovery plan:** PSRAM/RTC allocation, full compressor state/buffers, ABI, audio timing and USB/DIN/I2S behavior; never flash these emulator-only images to physical Protein.
4. **Physical audition and future Flavour macro design:** owner will test on physical Protein to choose preferred threshold, ratio law, output gain, attack, release, AutoGain and lookahead. Consider a true bypass at 0%, preset backwards compatibility and alternate linked-parameter curves.
5. **Expand feature test matrix:** Play/Arp mode latch transitions, MIDI, multiple FX settings, long playback, CPU load, clip statistics, audio matching, patch storage and NVS persistence.
6. **Deferred work:** Windows custom-wavetable importer; assign reserved Shift+Wave; remaining Round-105 research caveats as needed.

**Historical note:** The factory Flavour parameter is saved and displayed, and stock static analysis has not found an active DSP consumer. Factory presets contain varied Flavour values; this does not prove a stock audible effect. The custom compressor intentionally supplies a new consumer. A controlled original-firmware audio test was not completed, so the assertion that factory Flavour is completely inaudible remains high-confidence but not definitive.



## Stage 11 - hardware-candidate audit (12 November 2026)

Static audit of the Stage 10 image for physical-hardware use. The user reported Stage 10 working in the emulator. Full details, addresses and the flash/recovery procedure are in `Handoff.md` section 12.

- **Finding (probable root cause of left-only artifacts, Stages 4-10):** the pre-limiter hook `call8 0x42054a00` at 0x4200f29d clobbers `Fx__process` a9, which is live as the left pointer for limiter group 3 (`mov.n a9,a10` at 0x4200ea92, a10 = [a1+152] = left+32; used by the gain loop at 0x4200f561-0x4200f56d). Left samples 8-11 therefore skip limiter gain whenever the limiter acts. Fixed by reloading a9 from [a1+152] at 0x4200f550/0x4200f556. Needs a listening test to confirm.
- Compressor state relocated to internal DRAM (heap_caps_calloc INTERNAL|8BIT), with RTC 0x50000100 fallback; pointer cell 0x500000fc; null-pointer passthrough.
- Custom strings/tables relocated from IBUS 0x4205xxxx to RTC data 0x50000540-0x500005bf.
- Verified: emulator panel/mailbox stubs absent from the UF2; stock RTC use limited to 0x50000000-0x5000001f (`esp_sleep` sub-mode ref counts); IROM still within MMU entries 0-5; image checksum/SHA valid; Part__buildEffects runs once.
- Output: `PROTEIN_STAGE11_HW_CANDIDATE.UF2`, SHA-256 `e51d6c3e1f3b5d90c4e7ce8ecbaa909e7adcd0ae3ff14cfedb016cd95f2b13a7`. Static validation only.

## Stage 12 - envelope order AMP -> FILTER -> WAVE (12 November 2026)

User request; no other behavior changed. Selector byte 0x50000026 (1 WAVE, 2 FILTER, 3 AMP), single writer at 0x4205444b. Boot default byte 1 -> 3; branch at 0x42054427 `bnez` -> `beqz` swaps the Select cycle direction. Output `PROTEIN_STAGE12_HW_CANDIDATE_AMP_FIRST.UF2`, SHA-256 `4074f1f32249b7be7f043b50250bb0a6d89fe244588b679db02d46ce9eb25d0b`. Static validation only.

## Stage 13 - LFO encoder press fix (12 November 2026)

Stage 5's trampoline 0x42054b00 was hooked only through FrontPanel vtable slot 0x3c0732f4, but every observed caller of `FrontPanel__onEvent` calls 0x4201e640 directly (stock dispatcher 0x4201d7fd; custom wrapper 0x420539f8 via literal 0x420539f4). This is a correction to the Stage 5 "user-observed working" record: the user reported on 2026-11-12 that presses do nothing. Fix: literal 0x420539f4 -> 0x42054b00. Output `PROTEIN_STAGE13_HW_CANDIDATE_LFO_PRESS.UF2`, SHA-256 `87b1a3862e07631cbc9e8e279f9d3d7c4a000f7466f99d5218eadbb69c9d9da1`. Static validation only.

## Stage 14 - Flavour compressor attack 15 ms (12 November 2026)

Attack coefficient (state +0x5c) set once in `hw_init` to powf(0.1, 1/(0.015 x 11025)) = 0.98617309 (`0x3f7c75d7`); previously 0 (instant) from `Compressor__init`. Same stock time convention as the 200 ms release literal (verified: formula reproduces 0x3f7fbb99). `hw_init` 0x42054c1c, `hw_stereo` 0x42054c60. Output `PROTEIN_STAGE14_HW_CANDIDATE_ATTACK15MS.UF2`, SHA-256 `f38535fbf30f8c52a0ac9ea9e30a59c79cb889441cc030d5f49e2ca1b2b9b8e2`. Static validation only.

## Stage 15 - attack 30 ms, release 300 ms (12 November 2026)

Attack literal 0x42054c18 -> `0x3f7e3958` (30 ms); release literal 0x42054848 -> `0x3f7fd264` (300 ms); stock time convention powf(0.1, 1/(t x 11025)). Output `PROTEIN_STAGE15_HW_CANDIDATE_ATK30_REL300.UF2`, SHA-256 `502a3145b4a4d8e8738cbff1ce6b37332ff701db8e8fc94e61b08f63187cf948`. Static validation only.

## Stage 16 - boot splash "Version: Bane" (12 November 2026)

Splash format literal 0x420011a4 (only user: `l32r` at 0x42019395 in Boot__showSplash) repointed from 0x3c065e08 `Version %d.%02d b%d` to RTC 0x500005bf `Version: Bane`. The Info page literal 0x42001834 and the SysEx version reply are unchanged. Output `PROTEIN_STAGE16_HW_CANDIDATE_SPLASH_BANE.UF2`, SHA-256 `5cfa664463f9134508bdef5e995c82075975be3882206041093e72dabe1c0c83`. Static validation only.

## Functions map rebuilt from the Stage 16 code (12 November 2026)

`Functions_Map.csv` was rebuilt from the Stage 16 image itself, by recursive-descent disassembly from every hook and a byte comparison with the factory `PROTEIN.UF2` (SHA-256 `ffcd702a...4532`). Earlier docs were used only as a cross-check. Every changed or added byte belongs to exactly one patch row; there are no overlaps.
- Rows: 2,402 stock rows unchanged in content, 18 of them with an appended `[Stage 16: ...]` note (modified bytes or new callers), plus 79 patch rows: 13 modified stock sites/data, 36 live custom code/data rows, 16 `PATCH_DEAD` rows (unreferenced custom code/data) and 14 RTC data rows.
- Correction to the 2026-11-06 handoff: the NoteProcessor set-param vtable slot 0x3c070150 is not redirected (still 0x42013f40). The custom set-param hook 0x42053818 is unreferenced. Only note-on (0x3c070124) and note-off (0x3c070128) are redirected.
- Dead custom code still in the image: 0x42053818 (old set-param hook), 0x4205398f-0x420539df (old text-only envelope cycler), 0x42054800 (Stage 1-10 init), 0x42054900 (Stage 4-10 stereo process), plus the IROM string and table copies superseded in Stage 11.
- Correction to the Stage 12 note: `CustomADSR__selectHook` receives the event type (always 3 on its active path), not the encoder direction. The envelope order is therefore fixed (AMP -> FILTER -> WAVE) whichever way Select turns.
- Static findings, kept on backburner by the owner: (a) `CustomADSR__labelDraw` 0x42054680 does not forward Text__draw's stack inverse flag (it reads the wrapper's uninitialised [a1+0]), so the envelope name may sometimes render inverted; (b) the factory dispatcher routes a Select turn to the shown EnvGroup before the hook runs, so the turn may also change the last-edited stage by 0.01 per detent. Neither has been observed at runtime.

## Stage 17 - selectable latch count (13 November 2026)

Latch rewritten in C and appended at 0x42055020 (`latch_note_on` 0x420550b4, `latch_note_off` 0x42055290, `latch_ui_button` 0x42055340); NoteProcessor vtable slots 0x3c070124/0x3c070128 and the UI__onButton call at 0x4201d96f repointed. RTC 0x50000020 = capacity 0-4. Press cycle Off -> Latch 1 -> 2 -> 3 -> 4 -> Off with notices "Latch 1-4" / "Latch Off" from RTC 0x500005d0-0x500005f9.
- Corrections found while reading the old latch: (1) `Latch__captureHeldNotes` 0x42053488 keeps the 4 OLDEST held notes by note-on time, not the newest (the map row written on 2026-11-12 said "most recent"); (2) the eviction fragment 0x42054200 stores at a1+32..44 inside the note-on hook's window-overflow save area, so a spill during the eviction note-off could corrupt state. Both are retired with the old code.
- Host test of latch.c with a mock NoteProcessor passed all scripted cases. Output `PROTEIN_STAGE17_HW_CANDIDATE_LATCH_COUNT.UF2`, SHA-256 `dedc499300ccabeb02d72cf780ed3f57117646ee02f22bfcfdccd8764e4ff8e1`. Not yet emulator- or hardware-tested.

## Stage 18 - Flavour compressor: AutoGain off, 4 ms attack, smoothing (13 November 2026)

AutoGain byte +0x80 = 0 and attack `0x3f72f9fa` (4 ms) set in `CustomCompressorHW__init` 0x42054c24. Flavour is smoothed per block in `CustomCompressorHW__stereoProcess` 0x42054c6c (k 0.017977, 20 ms; RTC float 0x500000f8). Output gain mapping unchanged (0 to -8 dB). Output `PROTEIN_STAGE18_HW_CANDIDATE_FLAVOUR_SMOOTH.UF2`, SHA-256 `53a8ef540135748d1ed83ee866e6a179f90a83fac4c91921883e6609bb910293`. Static validation only.

## Stage 19 - Flavour output gain 0 to +15 dB (13 November 2026)

Mapper slope literal 0x42054844 -> 0xbe800000 (-0.25): output gain +15 x Flavour dB (was -8 x). Output `PROTEIN_STAGE19_HW_CANDIDATE_OUTGAIN_PLUS15.UF2`, SHA-256 `30312da7d7fc2ea7248e5ff91541c2d65a19a87469e6ebeb386908de7e8e3d7a`. Static validation only.

## Correction and Stage 20 - Flavour IS used by the factory voices (13 November 2026)

Corrects Round 15 ("no consumer"), Round 33 ("closed enumeration") and Round 45 ("part+0x10 stays null, q = 0"), and the same claims repeated in this project on 12-13 November. Found after the owner reproduced per-note pitch/timbre changes at high Flavour on the factory UF2 in the emulator.
- The "part" whose +0x10 holds the q pointer is the shared VoiceEngine object (base 0x3fcbdcc4, stride 0x4074). Its vtable 0x3c071148 slot +0x14 is the setter 0x4204e8a0 (`s32i.n a3, a2, 16`). `fcn.4201476c` (vtable 0x3c07020c) stores its argument at manager+0xcac and calls that setter on manager+4. `ParamMgr__loadDefaults` 0x4201584c calls `fcn.4201476c` for layers 0-3 with engine+0x32ec, the engine parameter array. So q = entry 20 = Flavour (engine id 0x14). Round 45 searched only for stores through pointers loaded from offsets 4 or 8 in engine/voice code and missed this virtual setter.
- Readers (each null-checked): Voice__trigger 0x4037b645, Voice__render 0x4037bfcb, VoiceEngine__controlTick 0x4037e74f. Effects as in Round-series notes for q, now confirmed live: per-note random wavetable scan offset (osc 1 and 2, up to about 25 q), osc 1 pitch offset (up to 0.1 q), cutoff offset (up to 0.1 q); tick jitter when q > 0.25; envelope stepping once per 8 ticks when q >= 0.5.
- Stage 20 disables it: 0x4204e8a3 `s32i.n a3,a2,16` -> `nop.n`. Output `PROTEIN_STAGE20_HW_CANDIDATE_NO_FLAVOUR_RANDOM.UF2`, SHA-256 `e936603a29e33b5e1c056006527b140d9b10771329b928aed71ad517f6478d2c`. Static validation only.
- Lesson: an address/offset scan cannot rule out reads through stored array pointers; check for setters that publish whole parameter arrays.

## Stage 21 - aggressive Flavour compressor curve (13 November 2026)

`flavour_map` (C, 0x420555a0) replaces the asm mapper: threshold -50x dB, ratio 1 + 99x^3, output gain 50x(1 - 1/R) - 6 sqrt(x) dB (OutGain ramp v may exceed 1; Compressor__process applies -30 + 60v without a clamp and Db__toLinear covers up to +100 dB, checked), attack 4 ms x 0.025^x, release 300 ms x 0.05^x, recomputed when x moves > 0.0005. __divsf3 resolved to ROM 0x40002274 (ESP-IDF 5.5.1 esp32s3.rom.libgcc.ld). Output `PROTEIN_STAGE21_HW_CANDIDATE_FLAVOUR_DESTROY.UF2`, SHA-256 `e867f9a47ada22c728c6307fb44dad8edd9de4a0f36a8bebc2f239b8936141f9`. Static validation only.

## Stage 22 - Flavour release 300 -> 100 ms (13 November 2026)

`flavour_map` release = 300 ms x (1/3)^x (literal 0x42055580 = 0.33333334). Output `PROTEIN_STAGE22_HW_CANDIDATE_REL300_100.UF2`, SHA-256 `491e81f276e2f368a47e7d04b77a09da0dd702248a934b17cd6292a6383618de`. Static validation only.

## Stage 23 - own C Flavour compressor (13 November 2026)

`fcomp_block` (fcomp.c, 0x420557a4, 0x428 bytes from 0x42055724) replaces the call to `CustomCompressorHW__stereoProcess` at 0x42054a39; the stock `Compressor__process` is no longer used for Flavour. Reason: stepped gain (every 4 samples) and a positive-peak-only detector produced inharmonic noise on decaying high notes. fcomp uses a two-polarity peak detector held over 40 samples, dB-domain attack/release smoothing, per-sample linear gain ramps and a 40-sample lookahead delay. Same curves as Stage 22. State: 616 bytes inside the block published at 0x500000fc, guarded by magic 0x464c4156. Only external calls: powf 0x4203d85c (on parameter change) and ROM __divsf3 0x40002274. Output `PROTEIN_STAGE23_HW_CANDIDATE_CUSTOM_COMPRESSOR.UF2`, SHA-256 `507fd07e22e2ed0863820b7b718352f5bd20198ec615bace11c663ea68e9dcc3`. Static validation only.
