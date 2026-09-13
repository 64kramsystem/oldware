# Alter Ego C64 data-format reference

## Scope and confidence

This document describes the data actually consumed by the Commodore 64
edition of *Alter Ego*. It is intended both as a preservation record and as
an implementation specification for a remake. The byte-level source of truth
is the twelve D64 images and the extracted REL streams under `inputs/alter_ego/`; the
behavioral source of truth is the resident program captured at the first name
prompt in `inputs/alter_ego/snapshots/alter_ego_male_first_interaction_ram.bin`.

The supplied male and female Disk 1 program files are byte-identical. Their
narrative MAP files are not: each edition has its own twelve-stage corpus
distributed across six logical MAP files. The interchange export therefore
uses one resident runtime model and preserves the edition on every disk-based
record.

The exporter in `tools/extract_interchange.py` asserts all boundaries rather
than attempting best-effort recovery. A changed marker, impossible pointer,
truncated record, bad LZW code, malformed descriptor, unexpected trailer, or
changed corpus total stops the export.

## Artifact layers

There are four distinct layers:

1. A Commodore DOS REL file as stored on disk.
2. A framed logical stream made from 254-byte host-side REL records.
3. A MAP section containing route metadata and a directory of compressed
   scene blocks.
4. Decompressed interaction descriptors containing routes, conditions,
   effects, choice dimensions, a presentation script, and text.

Do not conflate a disk block, a REL record, a compressed scene block, and an
interaction descriptor. They have different boundaries and identifiers.

## REL host extraction

The preserved `.rel` files are exact host-side extractions. Every record is
254 bytes:

| Offset | Length | Meaning |
| ---: | ---: | --- |
| 0 | 1 | leading `$3A` marker |
| 1 | 252 | logical MAP payload |
| 253 | 1 | trailing `$3A` marker |

Consequently:

```text
payload = concatenate(record[1:253] for every record)
```

The size of a valid extraction is divisible by 254, and both markers must be
present in every record. The two marker bytes are container framing, not MAP
content.

The live C64 reader sees a related 256-byte drive record at
`$8045-$8144`:

| Address | Length | Meaning |
| --- | ---: | --- |
| `$8045` | 1 | status/check byte; must be zero |
| `$8046` | 1 | record identifier |
| `$8047` | 1 | leading `$3A` marker |
| `$8048-$8143` | 252 | logical payload |
| `$8144` | 1 | trailing `$3A` marker |

The native routine at `$5C3D` converts a 32-bit payload offset to
`record = offset / 252` and `buffer_index = offset % 252 + 3`. `$5CE6`
copies payload bytes and rolls over at buffer index 255. `$5EAD` performs the
inverse conversion. The CIA 2 receive loop at `$5DAF` applies `EOR #$06` to
the sampled serial line code; this corrects the custom transfer encoding and
is not content encryption.

The twelve MAP streams contain 6,724 framed records in total.

## MAP files and life-stage sections

Each edition has `map1`, `map27`, `map3`, `map4`, `map5`, and `map6`.
`map27` contains two independent sections, one for stage 2 and one for stage
7. The other files contain one section:

| File | Section base | Stage |
| --- | ---: | ---: |
| `map1` | unsigned little-endian dword at payload offset 0 | 1 |
| `map27` | 4 | 2 |
| `map27` | unsigned little-endian dword at payload offset 0 | 7 |
| `map3` | 0 | 3 |
| `map4` | 0 | 4 |
| `map5` | 0 | 5 |
| `map6` | 0 | 6 |

Both `map1` streams have an absolute-load preamble between the four-byte
section offset and the stage-1 section. Each object is
`{uint16_le load_address, uint16_le length, uint8 data[length]}`:

| Object | Load | Male bytes | Female bytes | Role |
| ---: | ---: | ---: | ---: | --- |
| 0 | `$E800` | 4,560 | 4,560 | shared graphics resources |
| 1 | `$E000` | 1,532 | 1,610 | edition-specific resident text |
| 2 | `$8466` | 5,420 | 5,420 | questionnaire overlay |
| 3 | `$8466` | 18,436 | 18,436 | gameplay overlay |

The fourth object overwrites the questionnaire phase at the same logical
address. This is why the first-name RAM capture cannot authoritatively
describe later gameplay code; the GAR and listing cover both occupants.

Section-relative offsets below are measured from the section base:

```c
struct MapSection {
    uint16_le age_low;                 // +0
    uint16_le age_high;                // +2
    uint8     stage_config[52];        // +4
    uint16_le route_count;
    uint16_le route_words[route_count];
    uint16_le initial_cursor_cell_index;
    uint16_le terminal_scene_id;
    uint16_le block_span[terminal_scene_id];
    uint16_le scene0_decompressed_length;
    uint16_le scene0_compressed_length;
    uint8     blocks[];
};
```

`terminal_scene_id` is exclusive for the `block_span` array: legal entries
are `0 .. terminal_scene_id-1`. A zero span means that the scene ID has no
compressed block. A nonzero span is the complete byte size of that scene's
boundary table plus its compressed bytes. Starting at `blocks`, accumulate
the spans in scene-ID order to find every block. This directory is
authoritative; searching for plausible boundary tables misses valid blocks.

The variable route table initializes a prefix of the 90-word, 30-row by
3-column life-map grid persisted at `$81F7-$82AA`; the remaining cells are
cleared. `route_count` is always divisible by three. The initial cursor index
is a zero-based flattened cell index; the gameplay overlay divides it by
three for the initial row and uses the remainder as the initial column. A
word has this exact form:

```text
bits  0–7   global scene ID
bits  8–11  1-based icon selector; zero means no icon
bits 12–15  four-direction connector mask
```

Scene ID zero is an empty or connector cell, not an end-of-list sentinel.
IDs 1 through `terminal_scene_id-1` select the corresponding compressed
scene block. The unique cell whose ID equals `terminal_scene_id` is the
non-loadable end-of-stage marker and has no block span.

The last 48 bytes of `stage_config` are copied to persisted map/progression
state at `$81C5-$81F4`. They contain eight two-byte life-choice slot records
followed by eight corresponding 32-bit remaining-episode masks. The first
four transient bytes are loaded at `$803F-$8042`:

```text
+0  uint16_le  age_advance_scale_q10
+2  uint16_le  reverse_stage_index_8_minus_stage
```

The gameplay helper at `$91CD` advances time from the selected scene's
allocated compressed span. For an ordinary map scene, `span` is the exact
two-byte block span returned by `$982A`; for a side choice it is first divided
by three. The helper then computes:

```text
fractional_year_delta =
    (span * age_advance_scale_q10 + 512) >> 10
```

The result is a 16-bit fractional-year increment. Overflow past `$FFFF` adds
whole years to age at `$7F74` and, when applicable, child age at `$7F8C`;
the low 16 bits remain at `$7FC4`. The LIFE TIME screen converts that
fractional remainder to displayed months. The second word is exactly
`8 - stage`.

The gameplay overlay dispatches these selector kinds:

| Value | Choice |
| ---: | --- |
| 0 | unused slot |
| 1 | LIFE TIME |
| 2 | major purchase |
| 3 | college |
| 4 | relationships |
| 5 | life-status screen |
| 6 | family |
| 7 | later-life risk variant |
| 8 | marriage |
| 9 | adolescent risk variant |
| 10 | work |
| 11 | high school |

The first record byte is preserved as `scene_cursor_or_control`, because the
LIFE TIME and status handlers do not interpret it as an ordinary scene ID.

The final two header words are not generic trailers. In every one of the
fourteen sections they equal scene 0's final boundary and the exact minimum
compressed prefix required to produce it. Scene 0's allocated span contains
four or five further lookahead/padding bytes. The export distinguishes its
declared compressed stream from those allocation bytes.

The section bounds found in the corpus are:

| Edition | Map/section | Ages | Nonempty blocks | Descriptors |
| --- | --- | --- | ---: | ---: |
| female | map1/stage 1 | 0–3 | 34 | 758 |
| female | map27/stage 2 | 4–12 | 55 | 800 |
| female | map3/stage 3 | 13–18 | 117 | 1,295 |
| female | map4/stage 4 | 18–30 | 82 | 1,389 |
| female | map5/stage 5 | 31–45 | 66 | 1,297 |
| female | map6/stage 6 | 46–64 | 62 | 1,251 |
| female | map27/stage 7 | 65–90 | 34 | 610 |
| male | map1/stage 1 | 0–3 | 34 | 742 |
| male | map27/stage 2 | 4–12 | 55 | 734 |
| male | map3/stage 3 | 13–18 | 119 | 1,342 |
| male | map4/stage 4 | 18–30 | 85 | 1,326 |
| male | map5/stage 5 | 31–45 | 66 | 1,308 |
| male | map6/stage 6 | 46–64 | 64 | 1,259 |
| male | map27/stage 7 | 65–74 | 34 | 615 |

The female stage-7 upper age is 90 in the data. It is not normalized to the
male value.

Across both editions the authoritative directories contain 907 nonempty LZW
blocks and 14,726 descriptors.

## Compressed scene block

A nonempty `block_span` points to this structure:

```c
struct CompressedScene {
    uint16_le boundary_count;
    uint16_le boundary[boundary_count];
    uint8     compressed[];
};
```

The first boundary must be zero and all following boundaries must increase.
The last boundary is the exact decompressed byte count. Adjacent pairs select
descriptors:

```text
descriptor[i] = decompressed[boundary[i] : boundary[i + 1]]
descriptor_count = boundary_count - 1
```

The compressed portion ends at `block_start + block_span`; it does not run to
the next signature-looking byte sequence. The retained `block_spans` are also
used by the runtime's age-advance calculation.

### Grouped 9/10-bit LZW

This is an LZW-family dictionary with literals 0–255 and dictionary codes
256–1023. There are no clear or end codes. The required decompressed length
from the boundary table ends decoding.

The unusual part is code packing. Codes are not one continuous bitstream:

- At width 9, read nine source bytes and split them most-significant-bit first
  into eight 9-bit codes.
- The dictionary begins at code 256.
- After dictionary entry 256 has been added, `next_code` becomes 257. Before
  reading the following code, discard the six unused codes from the current
  9-bit group, change to width 10, and refill. Equivalently, switch when
  `width == 9 && next_code > 256`.
- At width 10, read ten source bytes and split them into eight 10-bit codes.
- The dictionary stops growing at 1,024 entries.

A continuous-bitstream decoder will often produce a plausible prefix and
then corrupt the remainder. The group discard at the width transition is
mandatory and is directly implemented by native helpers `$A850` and `$A974`.

Standard LZW reconstruction applies:

1. The first code must be a literal.
2. Emit the previous phrase.
3. For each later code, expand its prefix chain.
4. If `code == next_code`, use the previous phrase followed by its first
   byte (the conventional KwKwK case).
5. Add `previous_phrase + first_byte(current_phrase)` if the dictionary is
   not full.

At the first captured interaction, the male `map1` section resolves global
scene 26 to:

- count offset: decimal 89,016 (`$15BB8`);
- compressed data offset: 89,098;
- local descriptor index: 31;
- decompressed bytes 6,032–6,425 inclusive;
- descriptor length: 394 bytes.

Those 394 bytes match physical RAM at `$C2A2-$C42B`. The generated LZW prefix
entries 256–1023 match `$B77B-$BD7A`, and the suffix entries match
`$BE7B-$C17A`. This independently proves the container directory, code
packing, dictionary algorithm, and descriptor slicing.

## Interaction descriptor

Every one of the 14,726 descriptors parses with this grammar:

```c
struct Descriptor {
    uint8 route_count;
    uint8 routes[route_count];

    uint8 condition_count;
    Condition conditions[condition_count];

    uint8 effect_count;
    Effect effects[effect_count];

    uint8 choice_dimension_count;
    uint8 choice_radices[choice_dimension_count];

    uint8 script_length;
    uint8 script[script_length];

    uint16_le text_boundary_count;
    uint16_le text_boundary[text_boundary_count];
    uint8 text_pool[text_boundary[text_boundary_count - 1]];

    uint8 optional_zero_trailer[0 or 1];
};
```

The first text boundary is zero. Adjacent boundary pairs select text
fragments, so the number of fragments is `text_boundary_count - 1`.
Boundaries are nondecreasing because empty strings are legal.

Each exported descriptor has a stable human-readable ID:

```text
<edition>/<map>/stage-<n>/scene-<three digits>/chunk-<three digits>
```

For example:

```text
male/map1/stage-1/scene-026/chunk-031
```

The export records the semantic model and stable ID; the retained REL inputs
remain authoritative for raw bytes.

### Routes and mixed-radix choices

The `routes` array is the descriptor's result table. A completed set of user
choices is folded into a zero-based mixed-radix index:

```text
accumulator = 0
for each choice dimension i:
    accumulator = accumulator * choice_radices[i] + selection[i]
result_index = accumulator
next_route = routes[result_index]
```

Selections and route indexes are zero-based. There is no extra reserved route:
for every exported descriptor, `len(routes)` equals the product of
`choice_radices`, treating an empty product as one. Thus dimensions `[2, 3]`
map six response combinations to exactly six consecutive route entries.

### Conditions

A condition is five bytes:

```c
struct Condition {
    uint8    state;
    int16_be inclusive_low;
    int16_be inclusive_high;
};
```

The state vector has 64 little-endian 16-bit words at `$7F40-$7FBF`. The
condition succeeds when the addressed signed value lies inside the inclusive
range. Ordinary conditions are ANDed in their stored order and stop on the
first failure.

State `$FF` is not an unconditional condition. It separates two evaluation
phases:

- Conditions before `$FF` are `entry` conditions. The interpreter evaluates
  them before accepting the descriptor; failure rejects the descriptor.
- Conditions after `$FF` are `deferred` conditions. Presentation opcode 6
  evaluates them later, stores option 0 on success or option 1 on failure in
  choice dimension zero, and route selection uses that binary result.

All 1,669 descriptors containing `$FF` contain it exactly once and execute
opcode 6 exactly once. Every one declares the single radix `[2]`. The other
13,057 descriptors contain neither the separator nor opcode 6. The export
marks ordinary records with `evaluation_phase`, marks `$FF` as
`record_kind: "deferred_separator"`, and provides the entry/deferred counts
and separator index.

The two bound words are big-endian even though resident pointers and saved
state are little-endian. This is a property of the content record, not an
exporter convention.

### Effects

An effect is four bytes:

```c
struct Effect {
    uint8    state_index;
    uint8    operation_flags;
    int16_be operand;
};
```

`operation_flags & $40` means that the unsigned 16-bit operand is a state-word
index rather than a literal. Resolve it immediately before this effect:

```text
effective_operand =
    state[operand_unsigned] if operation_flags & $40
    else signed16(operand)
```

All 215 indirect records reference indices 0–63. The referenced word is read
from the live state produced by preceding effects in the same descriptor, so
effect order is observable. Its 16-bit bit pattern is then used by the
selected arithmetic operation; assignment copies it exactly, while add and
subtract use the runtime's signed 16-bit arithmetic behavior.

The operation code is the remaining value (`operation_flags & $BF` in the
original implementation). Values observed in the corpus are:

| Code | Interchange name | Semantics |
| ---: | --- | --- |
| 0 | `assign` | replace the state word with the operand |
| 1 | `reserved_no_arithmetic` | accepted by the runtime; no corpus record currently uses it |
| 2 | `add` | add the signed operand |
| 3 | `move_toward_100_percent` | move a trait toward 100 by the encoded proportion |
| 4 | `subtract` | subtract the signed operand |
| 5 | `proportional_decrease` | reduce a trait by the encoded proportion |
| 6 | `random_modulo` | produce a bounded random result |

Trait indices 0–11 receive percentage-aware handling in operations 3 and 5.
The export records either a literal operand or an indirect state index.

### Presentation script

The script is a compact bytecode separate from the resident execution VM:

| Opcode | Operands | Exported name | Role |
| ---: | ---: | --- | --- |
| 0 | 0 | `begin` | begin/reset interaction presentation |
| 1 | 1 | `render_text` | render text fragment by index |
| 2 | 2 | `define_choice_and_render` | packed choice coordinate, then text index |
| 3 | 1 | `present_text_panel` | render the indexed text fragment, terminate the current display-item list, and present it through the modal UI |
| 4 | 1 | `packed_choice_control` | declare one compact choice radix |
| 5 | 0 | `finish_selection` | finish the current selection sequence |
| 6 | 0 | `evaluate_deferred_conditions` | evaluate the deferred group and set binary choice dimension zero |

Opcode 5 is treated as a script terminator by the interchange view because
the resident parser transfers out of the presentation sequence there. Exactly
2,011 scripts contain one further byte; control flow never reads it. The
exporter accepts that byte but rejects any larger suffix or malformed opcode.

For opcode 2, the high nybble of the first operand is the zero-based choice
dimension and the low nybble is the zero-based option within that dimension:

```text
choice_dimension = packed_choice_metadata >> 4
choice_option    = packed_choice_metadata & $0F
```

Whenever a script uses opcode 2, these coordinates exactly enumerate every
option declared by `choice_radices`, once each. The exporter exposes both
decoded fields and rejects an out-of-range, duplicate, or missing coordinate.
Deferred-condition descriptors have radix `[2]` and routes but no opcode-2
presentation definitions: opcode 6 supplies their binary selection. Those
records are not mistaken for malformed option lists.

Opcode 4 is the compact alternative. Its operand is the descriptor's single
choice radix. All 543 descriptors that use opcode 4 have exactly one radix,
no opcode-2 definitions, and an opcode-4 operand equal to that radix (2–10).
The interchange exposes it as `choice_radix` and validates this invariant.

### Text encoding and substitutions

Descriptor text is mostly ASCII-compatible:

- `$00`: NUL control;
- `$0A`: newline;
- `$20-$7E`: literal text;
- `$80-$8B`: the twelve personality trait values;
- `$8C-$99`: resident-text lookups for state indices 12–25;
- `$9A`: numeric formatting of state 26, age in years.

The interchange `template` replaces dynamic bytes with names such as
`{calmness}`, `{occupation_id}`, and `{age_years}`. Unknown or nonprinting
bytes become `{BYTE_XX}` rather than being silently converted.

## State vector and save file

The mutable state vector is 64 unsigned little-endian words in RAM:

```text
state[i] at $7F40 + 2*i, 0 <= i < 64
```

Conditions and effects address it by index. Confirmed meanings are:

| Index | RAM | Meaning |
| ---: | ---: | --- |
| 0–11 | `$7F40-$7F57` | Calmness, Confidence, Expressiveness, Familial, Gentleness, Happiness, Intellectual, Physical, Social, Thoughtfulness, Trustworthiness, Vocational |
| 12 | `$7F58` | occupation ID |
| 13 | `$7F5A` | non-spouse partner ID |
| 14 | `$7F5C` | spouse ID |
| 15 | `$7F5E` | child-name ID |
| 16 | `$7F60` | romantic-meeting-location selector |
| 17–19 | `$7F62-$7F67` | paternal guardian, maternal guardian, and collective parents/guardians term IDs |
| 20–25 | `$7F68-$7F73` | partner trustworthiness, gentleness, calmness, happiness, confidence, and attractiveness intensity selectors |
| 26 | `$7F74` | age in years |
| 27–28 | `$7F76-$7F79` | cash, two base-1000 limbs |
| 29 | `$7F7A` | married flag |
| 30 | `$7F7C` | engaged flag |
| 31 | `$7F7E` | going-with/dating flag |
| 32 | `$7F80` | widowed flag (**inferred**: no descriptor, text token, or save fixture references this index; named from the marital-status run at 29-31) |
| 33–34 | `$7F82-$7F85` | debt, two base-1000 limbs |
| 36 | `$7F88` | current major-purchase category |
| 37 | `$7F8A` | gross annual salary in thousands of dollars per year; zero means no salary |
| 38 | `$7F8C` | child age in years |
| 39 | `$7F8E` | partner trustworthiness score; generated in the same third as selector 20 |
| 40 | `$7F90` | partner attractiveness score; generated in the same third as selector 25 |
| 43/44 | `$7F96/$7F98` | infidelity-attempt flag in male/female editions respectively; set after agreeing to pursue a second partner, consumed by the discovery/breakup branch, then cleared |
| 45 | `$7F9A` | male college-progress count; female number of children |
| 46 | `$7F9C` | male living-with-partner flag; female college-progress count |
| 47 | `$7F9E` | committed-relationship flag |
| 48 | `$7FA0` | male partner calmness score; generated in the same third as selector 22; unused by female descriptors |
| 49 | `$7FA2` | questionnaire-derived strict-parenting flag |
| 51 | `$7FA6` | male number of children; female child-sex ID |
| 55 | `$7FAE` | phase-reused school status code, detailed below |
| 35, 56–57, 59–63 | corresponding words | unused by every condition, effect, indirect operand, and dynamic text token in both corpora; preserved as reserved save words |

The full vector is saved at file offsets `$0000-$007F`. Even unnamed indices
are exported with their index, RAM address, save offset, type, and confidence.
Male state 44 is a released-content bug. The male relationship descriptor
tests state 44 for a positive value before the custody warning “And what about
the children?”, while its female counterpart tests the verified female child
count at state 45. The complete male corpus instead maintains its child count
at state 51 and never writes state 44; every supplied male save contains zero
there. Compatibility mode should preserve the erroneous state-44 read. A
corrected remake should test male state 51.

State 37's unit is fixed by both content and native arithmetic. The male job
menu prints annual salaries such as `$4000/yr.`, `$5000/yr.`, and `$6000/yr.`
beside assignments 4, 5, and 6. The gameplay routine multiplies state 37 by
elapsed years and retained-income percentage, then multiplies by 10 before
rounding to dollars: one stored unit is therefore `$1,000/year`, and the
factor 10 is `$1,000 / 100` percentage points. Values ending in `$500` are
coarsely authored table entries rather than produced by a runtime rounding
rule.

State 55 is a routing/status code rather than education progress. During
adolescence, zero means high school is not complete and one is a completion
latch that is cleared at the stage transition. In female college content,
zero means inactive or dropped out, one means enrolled, and two or more means
graduated. In male college content, a positive value means enrolled and zero
means inactive, dropped out, or graduated. College progress is stored
separately at male state 45 and female state 46, with 12 as the graduation
threshold.
The score names at 39, 40, and male 48 are verified from all partner-creation
paths: each starts with a random value 0–32, then adds 33 or 66 in lockstep
with the corresponding three-valued characteristic selector. State 47 is set
when the narrative says a pair becomes steady or otherwise committed, tested
before commitment-specific routes, and cleared on separation. Several later
indices deliberately have edition-specific names. For example,
the male corpus uses states 41/42 for substance-use and marital-stress
tallies, while the female corpus uses 42/43; child sex is state 50 in the male
edition and 51 in the female edition. The interchange schema records these as
`edition_names` rather than forcing a false common layout.

The save payload is 380 bytes:

| File offset | Length | RAM | Source/meaning |
| ---: | ---: | ---: | --- |
| `$0000` | 128 | `$7F40` | 64-word state vector |
| `$0080` | 48 | `$81C5` | persisted stage life-choice configuration |
| `$00B0` | 180 | `$81F7` | 30 × 3 × 2-byte life-map grid |
| `$0164` | 16 | `$82CB` | NUL-padded player name |
| `$0174` | 2 | `$82BD` | signed life-stage value |
| `$0176` | 2 | `$82AB` | acquisitions bitmask |
| `$0178` | 4 | `$7FC0` | used-family-episode bitmask |

This scatter/gather mapping is exact. In particular, file `$0178-$017B`
does **not** map to `$82AD-$82B0`: those resident bytes begin a life-map
coordinate-boundary table. Gameplay helper `$8DD0` subtracts this mask from
the family slot's available mask before selection and adds the selected bit
afterward. Other life-choice slots instead clear the bit directly in their
stage-local remaining mask.

The 48-byte stage configuration mirrors MAP `stage_config[4:52]`. It has a
verified structure of eight two-byte selector records
`{scene_cursor_or_control, selector_kind}` followed by eight little-endian
32-bit remaining-episode masks. Pristine masks are contiguous low-bit runs;
saved games clear bits as finite side experiences are consumed. The selector
enumeration is listed above.

The life-map grid uses the same word layout as the MAP route table described
above. Completed-stage saves clear an experience cell's icon-selector low
nybble while preserving its connector high nybble: the central experience
disappears, but the traversed path remains drawn.

Life-stage values are `-1` through `-7` for infancy, childhood, adolescence,
young adulthood, adulthood, middle age, and old age.

Acquisition bits 1–11 mean watch, stereo equipment, photo equipment, video
equipment, computer, sports equipment, wardrobe, automobile, library, boat,
and home. Bit 0 is not one of the displayed acquisitions.

Cash and debt are represented as two base-1000 limbs rather than a normal
32-bit binary number:

```text
value = low_limb + 1000 * high_limb
```

This matches the game's three-digit formatting and arithmetic helpers.

Each edition also supplies a 160-byte `AESD` directory file consisting of
five 32-byte display-string records. The provided corpus contains ten
380-byte `AES0`–`AES4` payloads plus the two directory files; these are
stage/checkpoint samples, not twelve empty templates. They are useful remake
fixtures because their life-map prefixes match the corresponding MAP section
word for word before consumed icon selectors are cleared.

## Resident text resource

Physical RAM under KERNAL ROM at `$E000` begins with fourteen little-endian
pointers. Each points to a null-word-terminated pointer list; those pointers
select NUL-terminated ASCII-compatible strings in the same hidden-RAM pool.
The categories are:

1. career options;
2. female-name list A;
3. female-name list B;
4. mixed personal names;
5. meeting locations;
6. paternal guardian names;
7. maternal guardian names;
8. guardian terms;
9. intensity scale 0;
10. intensity scale 1;
11. intensity scale 2;
12. intensity scale 3;
13. intensity scale 4;
14. intensity scale 5.

Category index `n` is selected by substitution byte `$8C+n` and state word
`12+n`. The state word is the zero-based string index within that category.
Thus `$8C`/state 12 selects a career, `$8D-$8F` select the three name lists,
`$90` selects a meeting phrase, `$91-$93` select guardian terms, and
`$94-$99` select the six characteristic intensities. `$9A` is different: it
formats state 26 as numeric age and has no fifteenth text category.
`resident.json` exports these bindings explicitly.

The runtime banks out KERNAL ROM when it needs this data. A CPU-visible dump
made with KERNAL enabled shows ROM instead and is therefore not a valid source
for the pool. The physical-RAM snapshot is the authoritative source.

## Personality questionnaire

The resident questionnaire has 25 statements, not 50. Its line-pointer table
at `$9398-$942D` contains exactly 75 little-endian pointers: three display
lines per question. `$9078` computes `table + question_index*6`, always
renders the first line, and suppresses a second or third line whose first byte
is NUL. Answers use `0 = unset`, `1 = TRUE`, and `2 = FALSE`.

Questions are shown in seven pages starting at indices 0, 4, 8, 12, 16, 20,
and 24; the page sizes are 4, 4, 4, 4, 4, 4, and 1. Before accepting the
questionnaire, `$8FBB` scans the 25-byte answer buffer and returns to the page
containing the first unset answer. Computer-selected profiles do not invent
scores directly: `$9323` fills each answer byte with
`(SID voice-3 oscillator/noise output & 1) + 1`.

The scoring tables are:

| Range | Records | Answer |
| --- | ---: | --- |
| `$94AE-$971E` | 25 × 25 bytes | TRUE |
| `$971F-$998F` | 25 × 25 bytes | FALSE |

Each scoring record is:

```c
struct QuestionnaireScoreRecord {
    uint8 operation_count;             // maximum observed: 8
    struct {
        uint8 operation;
        uint8 state_index;
        uint8 magnitude;
    } operations[operation_count];
    uint8 zero_padding[24 - 3*operation_count];
};
```

The operations are ordered and deterministic:

| Code | Effect |
| ---: | --- |
| 0 | `state[index] = magnitude` |
| 1 | `state[index] += floor((100-state[index]) * magnitude / 100)` |
| 2 | `state[index] -= floor(state[index] * magnitude / 100)` |
| 3 | `state[index] += magnitude` |
| 4 | same addition operation through a second dispatch arm |

Codes 3 and 4 are not used by the fifty TRUE/FALSE records. Scoring first
sets all twelve traits to 50 and state 49 to zero. It then applies the chosen
record for questions 1 through 25 in order. There is no normalization, clamp,
or random call in this routine. Questions affect traits 0–5 and 8–10;
Intellectual, Physical, and Vocational remain at 50 at this stage.

Question 22 TRUE—“My parents were extremely strict
disciplinarians”—is the sole questionnaire operation that assigns state 49,
setting it to one. This uniquely identifies that word as a strict-parenting
flag. A curious encoded no-op is also preserved: Question 7 TRUE contains a
zero-percent decrease of Gentleness.

The manual says identical answers are unlikely to produce identical
personalities. The scoring routine itself contradicts that explanation: it
is deterministic. The statement could still describe a later stage-entry
perturbation, so a remake should not add questionnaire randomness unless a
post-scoring writer is independently demonstrated.

Three resident menu records immediately precede the score tables:

```c
struct MenuDescriptor {
    uint16_le first_selectable_screen_row;
    uint16_le choice_count;
    struct {
        uint16_le text_pointer;
        uint16_le screen_row;
    } display_entries[];               // terminated by pointer=0,row=0
};
```

They occupy `$942E-$9455` (seven life phases), `$9456-$9491`
(new/resume), and `$9492-$94AD` (three personality modes).

## Interchange files

`interchange/maps.jsonl` has one object per edition/MAP file with the stage
ages, routes, initial cursor, life-choice slots, and block spans needed by the
age-advance calculation.

`interchange/descriptors.jsonl` has one object per descriptor. It contains
the parsed condition/effect/choice/script/text model.

`interchange/save_state.json` describes all 64 state slots and the remaining
save segments without inventing names for reserved or semantically neutral
fields.

`interchange/resident.json` exports all 25 questionnaire statements and both
score records, the three menu descriptors, and all fourteen resident
under-KERNAL text categories.

The JSONL files are intentionally line-oriented. A remake can stream or index
14,726 descriptors without loading a monolithic JSON document, and a textual
diff isolates one record.

## Compatibility guidance

A new engine should treat the exported JSON as immutable source data and use
stable descriptor IDs as primary keys. Preserve the original order of
conditions, effects, routes, script operations, and text fragments: the C64
runtime is ordered, and reordering apparently commutative effects can change
later indirect operations.

Use signed 16-bit arithmetic where the original does, and decide explicitly
whether the remake wraps or widens intermediate values. The C64 implementation
wraps naturally in several VM operations. Random choices should use an
injectable generator so recorded playthroughs remain reproducible.

Do not infer that every state score is morally “good” when high. The manual
describes paired continua, and the content deliberately tests both ends.
Similarly, do not normalize gender-specific text or age bounds during import;
keep source fidelity in the data layer and make modernization a separate,
reviewable adaptation.

## Resolved field semantics

The formerly neutral meanings of states 37, male 44, and 55 are resolved
above from paired edition content and native arithmetic. The historical
source-language identifier for male state 44 is unavailable, but the released
misindex and its intended child-count predicate are behaviorally complete.
