# MattiaOS: Un Sistema Operativo Generato dall'IA
*(Italian version. For the English version, scroll down.)*

Questo progetto è un sistema operativo completamente funzionante, la cui architettura e base di codice sono state generate interamente tramite **Intelligenza Artificiale**. Sviluppato in Italia come proof-of-concept, dimostra le straordinarie capacità di programmazione dei moderni modelli linguistici di grandi dimensioni, in questo caso Claude Sonnet 4.6 Extended.

Il risultato è un sistema operativo creato da un ragazzo di 14 anni e una AI, senza che una singola riga di codice fosse scritta manualmente da un autore umano.

---

## Metodologia di Sviluppo

L'intero processo di sviluppo — dal design del kernel fino alla generazione dell'immagine ISO avviabile — è stato gestito delegando i task all'IA.

- **Compilazione In-Chat:** I compilatori necessari sono stati forniti all'IA direttamente tramite l'interfaccia di chat, permettendole di compilare il codice sorgente in binari e assemblare l'immagine ISO in autonomia.
- **Toolchain:** Il sistema è stato compilato utilizzando **NASM 3.01** per i componenti Assembly e **GCC 13.3 Cross-Compiler** (`x86_64-elf-gcc`) per il codice C, garantendo l'assenza totale di dipendenze da librerie di sistema esterne.

## Architettura Tecnica

Il processo di boot segue una catena custom a due stadi:

```
BIOS -> stage1.asm (El Torito, 512B) -> stage2.asm (loader custom)
     -> patcha le Limine protocol responses in memoria
     -> kernel.elf (higher-half, 0xFFFFFFFF80000000)
```

Il kernel gira in Long Mode (64-bit) nella metà alta dello spazio degli indirizzi. La mappa fisica inizia a 0x200000. L'offset HHDM è 0xFFFF800000000000.

**Gestione della memoria**
- PMM: allocatore a bitmap, posizionato a `max(0x400000, kernel_phys_end)` per non sovrascrivere il BSS del kernel.
- VMM: gestione delle page table a 4 livelli.
- Slab allocator: 9 cache a dimensione fissa (16-4096 byte).

**Interrupt e CPU**
- GDT, IDT, gestione delle eccezioni.
- Remapping IRQ via 8259 PIC e I/O APIC.
- Timer LAPIC calibrato su PIT channel 2 (tick da 1ms).

**Driver**
- Tastiera PS/2: scan code set 2, layout italiano, Ctrl+C genera ETX.
- VGA text mode: 80x25, codepage CP437, cursore hardware, scrollback da 128 righe.
- PCI: enumerazione bus e dispositivi.
- AHCI (SATA), NVMe.
- VirtIO-Net e e1000 (schede di rete).
- Framebuffer GOP.

**Filesystem**
- VFS: mount table, risoluzione path, `open/close/read/write`, enumerazione directory in due fasi (entry reali + mount point sintetici).
- tmpfs: filesystem in RAM. Bug critico risolto: i dati scritti persistono correttamente tra sessioni `open/close` successive.
- devfs: montato su `/dev`, espone `null` e `zero`.
- FAT32 e MattiaFS: driver aggiuntivi.

**Rete**
- Stack completo: Ethernet, ARP, IPv4, ICMP, TCP, UDP, socket layer.

**Kernel e Syscall Personalizzati:** Sebbene il design del filesystem tragga ispirazione da Linux, il kernel e le chiamate di sistema (syscall) sono stati scritti da zero e sono interamente proprietari. A causa di questa architettura custom, l'IA ha sviluppato un set di comandi built-in per consentire l'interazione diretta con il sistema prima che il loader ELF (Phase 13) fosse implementato.

## Comandi Disponibili

La shell di sistema supporta attualmente i seguenti comandi nativi:

| Comando | Descrizione |
| :--- | :--- |
| `help` | Elenca e descrive tutti i comandi disponibili |
| `ver` | Mostra la versione attuale del sistema operativo |
| `mem` | Mostra le statistiche della RAM e i processi attivi |
| `pci` | Elenca i dispositivi PCI e le periferiche connesse |
| `ls [path]` | Elenca i file e le directory nel percorso corrente |
| `cd [path]` | Cambia la directory di lavoro corrente |
| `open [path]` | Pager full-screen per file testo, binari ed ELF |
| `edit [path]` | Editor di testo integrato (Ctrl+S per salvare, ESC per uscire) |
| `tree [path]` | Stampa la struttura completa dell'albero del filesystem |
| `clear` | Pulisce la schermata del terminale |
| `halt` | Esegue lo spegnimento sicuro del sistema via ACPI |
| `reboot` | Riavvia il sistema |
| `easteregg` | Sorpresa |
| `diag` | Esegue la diagnostica interna del VFS |

Funzionalita' della console:
- Frecce Su/Giu: navigazione nella history degli ultimi 32 comandi.
- Shift+Su: entra in modalita' scrollback (128 righe); Shift+Giu per uscire.
- Ctrl+C: interrompe i comandi in esecuzione (anche durante `tree` e `ls`).
- Layout tastiera italiano con codifica CP437 corretta.

## Layout del Filesystem

```
/
├── dev/                  (devfs)
│   ├── null
│   └── zero
├── etc/
├── home/
│   ├── ciao.txt
│   └── mattia_hello      (programma demo, eseguibile in Phase 13)
├── tmp/
├── var/
├── sys/
│   └── PATH              (registro comandi: nome=exe:dep1:dep2)
├── Binari_Eseguibili/    (equivalente di /bin, stub in attesa di Phase 13)
└── Dipendenze/
    └── libmattiac.so     (stub libreria standard)
```

## Build

**Requisiti**
- `x86_64-elf-gcc` 13.3 (cross-compiler)
- NASM 3.01
- GNU Binutils 2.42
- xorriso 1.5.6
- Host Linux (testato su Ubuntu 24.04)

```bash
make kernel   # compila solo il kernel
make iso      # build completa: kernel + bootloader + immagine ISO
```

Output: `mattiaos.iso` — immagine ISO avviabile in formato El Torito.

## Test su VirtualBox

Testato con le seguenti impostazioni:

| Impostazione | Valore |
| :--- | :--- |
| Tipo | Other / Other (64-bit) |
| Chipset | PIIX3 |
| Memoria | minimo 256 MB |
| Ordine di boot | Solo ottico |
| I/O APIC | Abilitato (obbligatorio) |
| EFI | Disabilitato (obbligatorio — solo boot BIOS legacy) |
| Controller storage | IDE (non SATA) |
| Video | VBoxVGA, 16 MB |
| HPET | Disabilitato |

## Bug Critici Risolti Durante lo Sviluppo

- **tmpfs write/read:** `tmpfs_write` aggiornava solo la copia locale del vnode (`node->size`), non il campo canonico dentro `tmpfs_node_t` (`n->vnode.size`). Ogni `vfs_open` produce una copia superficiale del vnode tramite `readdir`; la size della copia era sempre 0, rendendo tutti i file scritti a runtime apparentemente vuoti alla riapertura.
- **Ctrl+C:** `ascii_from_vk` controllava Ctrl+lettera dopo il check della lettera, quindi Ctrl+C produceva `'c'` invece di ETX (3). Risolto spostando il check Ctrl prima.
- **vfs_is_dir:** Usava `ops->readdir != NULL` come fallback per il rilevamento delle directory. Tutti i nodi tmpfs condividono la stessa ops table, quindi ogni file veniva riportato come directory.
- **Enter spurio da autorepeat:** Lo stato AR non veniva resettato all'Enter, causando pressioni ripetute di Enter dopo qualsiasi comando che impiegava più di 250ms.

## Roadmap (Phase 13+)

- ELF loader: `process_execve()`, `load_elf()`, stack iniziale con `argc/argv/envp/auxv`
- Syscall dispatcher: MSR_LSTAR + handler assembly + dispatch table
- Syscall prioritarie: `execve`, `rt_sigaction`, `setsid`, `getdents64`, `nanosleep`
- COW page fault handler
- Completamento della libc
- Userspace: init, shell, coreutils (sorgenti già presenti, in attesa del loader ELF)

---

> **Curiosita'**: Anche questo testo e' stato generato dall'IA (Claude Sonnet 4.6 Extended).

> **Note**:
> - L'output dei comandi e' in italiano.
> - La versione reale del progetto e' la 5.7, considerando tutte le sessioni di sviluppo. Su GitHub e' pubblicata come v1.0.
> - L'intero progetto e' documentato in dettaglio nel file `OS_PROJECT_BLUEPRINT.md`, generato dall'IA durante lo sviluppo. Si trova nello ZIP con i file sorgenti.

---
---

# MattiaOS: An AI-Generated Operating System
*(English Version)*

This project is a fully functional operating system whose architecture and codebase were generated entirely by **Artificial Intelligence**. Developed in Italy as a proof-of-concept, it demonstrates the extraordinary programming capabilities of modern Large Language Models, in this case Claude Sonnet 4.6 Extended.

The result is an operating system created by a 14-year-old boy and an AI, without a single line of code being written manually by a human author.

---

## Development Methodology

The entire development process — from kernel design to the generation of the bootable ISO image — was managed by delegating tasks to the AI.

- **In-Chat Compilation:** The necessary compilers were provided to the AI directly through the chat interface, allowing it to compile source code into binaries and assemble the ISO image autonomously.
- **Toolchain:** The system was compiled using **NASM 3.01** for the Assembly components and **GCC 13.3 Cross-Compiler** (`x86_64-elf-gcc`) for the C code, ensuring a total absence of external system library dependencies.

## Technical Architecture

The boot process follows a custom two-stage chain:

```
BIOS -> stage1.asm (El Torito, 512B) -> stage2.asm (custom loader)
     -> patches Limine protocol responses in memory
     -> kernel.elf (higher-half, 0xFFFFFFFF80000000)
```

The kernel runs in Long Mode (64-bit) in the upper half of the address space. Physical memory starts at 0x200000. The HHDM offset is 0xFFFF800000000000.

**Memory management**
- PMM: bitmap allocator, placed at `max(0x400000, kernel_phys_end)` to avoid overwriting the kernel BSS.
- VMM: 4-level page table management.
- Slab allocator: 9 fixed-size caches (16-4096 bytes).

**Interrupts and CPU**
- GDT, IDT, exception handlers.
- IRQ remapping via 8259 PIC and I/O APIC.
- LAPIC timer calibrated against PIT channel 2 (1ms tick).

**Drivers**
- PS/2 keyboard: scan code set 2, Italian layout, Ctrl+C produces ETX.
- VGA text mode: 80x25, CP437 codepage, hardware cursor, 128-line scrollback ring buffer.
- PCI: bus and device enumeration.
- AHCI (SATA), NVMe.
- VirtIO-Net and e1000 (network cards).
- GOP framebuffer.

**Filesystem**
- VFS: mount table, path resolution, `open/close/read/write`, two-phase directory enumeration (real entries + synthetic mount point entries).
- tmpfs: RAM filesystem. Critical fix applied: data written to files persists correctly across `open/close` cycles.
- devfs: mounted at `/dev`, exposes `null` and `zero`.
- FAT32 and MattiaFS: additional filesystem drivers.

**Networking**
- Full stack: Ethernet, ARP, IPv4, ICMP, TCP, UDP, socket layer.

**Custom Kernel and Syscalls:** While the filesystem design takes inspiration from Linux, the kernel and system calls (syscalls) were written from scratch and are entirely proprietary. Because of this custom syscall architecture, the AI developed a set of built-in commands for direct interaction with the system before the ELF loader (Phase 13) is implemented.

## Available Commands

The system shell currently supports the following native commands:

| Command | Description |
| :--- | :--- |
| `help` | Lists and describes all available commands |
| `ver` | Displays the current operating system version |
| `mem` | Shows RAM statistics and active processes |
| `pci` | Lists connected PCI devices and peripherals |
| `ls [path]` | Lists files and directories in the current path |
| `cd [path]` | Changes the current working directory |
| `open [path]` | Full-screen pager for text files, binaries, and ELF files |
| `edit [path]` | Integrated text editor (Ctrl+S to save, ESC to exit) |
| `tree [path]` | Prints the complete filesystem tree from the current directory |
| `clear` | Clears the terminal screen |
| `halt` | Executes a safe ACPI system shutdown |
| `reboot` | Reboots the system |
| `easteregg` | Surprise |
| `diag` | Runs internal VFS diagnostics |

Console features:
- Up/Down arrows: navigate the last 32 commands in history.
- Shift+Up: enter scrollback mode (128 lines); Shift+Down to exit.
- Ctrl+C: interrupts running commands (including `tree` and `ls`).
- Italian keyboard layout with correct CP437 encoding.

## Filesystem Layout

```
/
├── dev/                  (devfs)
│   ├── null
│   └── zero
├── etc/
├── home/
│   ├── ciao.txt
│   └── mattia_hello      (demo program, executable in Phase 13)
├── tmp/
├── var/
├── sys/
│   └── PATH              (command registry: name=exe:dep1:dep2)
├── Binari_Eseguibili/    (equivalent of /bin, stubs pending Phase 13)
└── Dipendenze/
    └── libmattiac.so     (standard library stub)
```

## Build

**Requirements**
- `x86_64-elf-gcc` 13.3 (cross-compiler)
- NASM 3.01
- GNU Binutils 2.42
- xorriso 1.5.6
- Linux host (Ubuntu 24.04 tested)

```bash
make kernel   # compile kernel only
make iso      # full build: kernel + bootloader + ISO image
```

Output: `mattiaos.iso` — bootable El Torito ISO image.

## Testing on VirtualBox

Tested with the following settings:

| Setting | Value |
| :--- | :--- |
| Type | Other / Other (64-bit) |
| Chipset | PIIX3 |
| Memory | 256 MB minimum |
| Boot order | Optical only |
| I/O APIC | Enabled (required) |
| EFI | Disabled (required — BIOS legacy boot only) |
| Storage controller | IDE (not SATA) |
| Video | VBoxVGA, 16 MB |
| HPET | Disabled |

## Key Bugs Fixed During Development

- **tmpfs write/read:** `tmpfs_write` was updating only the local vnode copy (`node->size`), not the canonical field inside `tmpfs_node_t` (`n->vnode.size`). Every `vfs_open` produces a shallow copy of the vnode via `readdir`; the copy's size was always 0, making all runtime-written files appear empty on the next open.
- **Ctrl+C:** `ascii_from_vk` checked Ctrl+letter after the letter check, so Ctrl+C produced `'c'` instead of ETX (3). Fixed by moving the Ctrl check first.
- **vfs_is_dir:** Used `ops->readdir != NULL` as a fallback for directory detection. All tmpfs nodes share the same ops table, so every file was reported as a directory.
- **Autorepeat spurious Enter:** AR state was not reset on Enter, causing repeated Enter keypresses after any command that took longer than 250ms.

## Roadmap (Phase 13+)

- ELF loader: `process_execve()`, `load_elf()`, initial stack with `argc/argv/envp/auxv`
- Syscall dispatcher: MSR_LSTAR + assembly handler + dispatch table
- Priority syscalls: `execve`, `rt_sigaction`, `setsid`, `getdents64`, `nanosleep`
- COW page fault handler
- libc completion
- Userspace: init, shell, coreutils (source exists, awaiting ELF loader)

---

> **Note**: This text was written by Claude Sonnet 4.6 Extended.

> **Notes**:
> - The command output is in Italian.
> - The real version of the project is 5.7, counting all development sessions. It is published on GitHub as v1.0.
> - The entire project is documented in detail in `OS_PROJECT_BLUEPRINT.md`, generated by the AI during development. It is included in the ZIP file with the source files.
