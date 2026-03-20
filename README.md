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
> - Si prevede una traduzione in inglese.

---
---

# MattiaOS: An AI-Generated Operating System
*(Italian version. For the English version, scroll down.)*

This project is a fully functional operating system, whose architecture and codebase were generated entirely using Artificial Intelligence. Developed in Italy as a proof-of-concept, it demonstrates the extraordinary programming capabilities of modern large-scale language models, in this case Claude Sonnet 4.6 Extended.

The result is an operating system created by a 14-year-old boy and an AI, without a single line of code being written manually by a human author.

---

## Development Methodology

The entire development process—from kernel design to bootable ISO image generation—was managed by delegating tasks to the AI.

- **In-Chat Compilation:** The necessary compilers were provided to the AI ​​directly via the chat interface, allowing it to compile the source code into binaries and assemble the ISO image autonomously.
- **Toolchain:** The system was compiled using **NASM 3.01** for assembly components and **GCC 13.3 Cross-Compiler** (`x86_64-elf-gcc`) for C code, ensuring a complete absence of dependencies on external system libraries.

## Technical Architecture

The boot process follows a custom two-stage chain:

```
BIOS -> stage1.asm (El Torito, 512B) -> stage2.asm (custom loader)
-> patches Limine protocol responses in memory
-> kernel.elf (higher-half, 0xFFFFFFFF80000000)
```

The kernel runs in Long Mode (64-bit) in the upper half of the address space. The physical map starts at 0x200000. The HHDM offset is 0xFFFF800000000000.

**Memory Management**
- PMM: Bitmap allocator, set to `max(0x400000, kernel_phys_end)` to avoid overwriting the kernel BSS.
- VMM: 4-level page table management.
- Slab allocator: 9 fixed-size caches (16-4096 bytes).

**Interrupts and CPU**
- GDT, IDT, exception handling.
- IRQ remapping via 8259 PIC and APIC I/O.
- LAPIC timer calibrated on PIT channel 2 (1ms tick).

**Drivers**
- PS/2 keyboard: scan code set 2, Italian layout, Ctrl+C generates ETX.
- VGA text mode: 80x25, CP437 codepage, hardware cursor, 128-line scrollback.
- PCI: Bus and device enumeration.
- AHCI (SATA), NVMe.
- VirtIO-Net and e1000 (network cards).
- GOP Framebuffer.

**Filesystem**
- VFS: Mount table, path resolution, open/close/read/write, two-step directory enumeration (real entries + synthetic mount points).
- tmpfs: Filesystem in RAM. Critical bug fixed: Written data persists correctly between successive open/close sessions.
- devfs: Mounted on /dev, exposes null and zero.
- FAT32 and MattiaFS: Additional drivers.

**Network**
- Full stack: Ethernet, ARP, IPv4, ICMP, TCP, UDP, socket layer.

**Custom Kernel and Syscalls:** Although the filesystem design draws inspiration from Linux, the kernel and system calls (syscalls) were written from scratch and are entirely proprietary. Because of this custom architecture, the AI ​​developed a set of built-in commands to allow direct interaction with the system before the ELF loader (Phase 13) was implemented.

## Available Commands

The system shell currently supports the following native commands:

| Command | Description |
| :--- | :--- |
| `help` | Lists and describes all available commands |
| `ver` | Displays the current operating system version |
| `mem` | Displays RAM statistics and active processes |
| `pci` | Lists PCI devices and connected peripherals |
| `ls [path]` | Lists files and directories in the current path |
| `cd [path]` | Changes the current working directory |
| `open [path]` | Full-screen pager for text, binary, and ELF files |
| `edit [path]` | Built-in text editor (Ctrl+S to save, ESC to exit) |
| `tree [path]` | Prints the complete filesystem tree |
| `clear` | Clears the terminal screen |
| `halt` | Performs a safe ACPI shutdown |
| `reboot` | Reboots the system |
| `easteregg` | Surprise |
| `diag` | Runs internal VFS diagnostics |

Console features:
- Up/Down arrows: navigate through the history of the last 32 commands.
- Shift+Up: enters scrollback mode (128 lines); Shift+Down to exit.
- Ctrl+C: interrupts running commands (including during `tree` and `ls`).
- Italian keyboard layout with correct CP437 encoding.

## Filesystem Layout

```
/
├── dev/ (devfs)
│ ├── null
│ └── zero
├── etc/
├── home/
│ ├── hello.txt
│ └── mattia_hello (demo program, executable in Phase 13)
├── tmp/
├── var/
├── sys/
│ └── PATH (command log: name=exe:dep1:dep2)
├── Executable_Binaries/ (equivalent of /bin, stub waiting for Phase 13) 13)
└── Dependencies/
└── libmattiac.so (standard library stub)
```

## Build

**Requirements**
- `x86_64-elf-gcc` 13.3 (cross-compiler)
- NASM 3.01
- GNU Binutils 2.42
- xorriso 1.5.6
- Linux Host (tested on Ubuntu 24.04)

```bash
make kernel # compiles the kernel only
make iso # complete build: kernel + bootloader + ISO image
```

Output: `mattiaos.iso` — a bootable ISO image in El Torito format.

## Testing on VirtualBox

Tested with the following settings:

| Setting | Value |
| :--- | :--- |
| Type | Other / Other (64-bit) |
| Chipset | PIIX3 |
| Memory | minimum 256 MB |
| Boot Order | Optical Only |
| I/O APIC | Enabled (required) |
| EFI | Disabled (required — legacy BIOS boot only) |
| Storage Controller | IDE (not SATA) |
| Video | VBoxVGA, 16 MB |
| HPET | Disabled |

## Critical Bug Fixes During Development

- **tmpfs write/read:** `tmpfs_write` updated only the local copy of the vnode (`node->size`), not the canonical field in `tmpfs_node_t` (`n->vnode.size`). Each `vfs_open` produces a shallow copy of the vnode via `readdir`; The copy size was always 0, making all files written at runtime appear empty upon reopening.
- **Ctrl+C:** `ascii_from_vk` checked Ctrl+letter after the letter check, so Ctrl+C produced `'c'` instead of ETX (3). Fixed by moving the Ctrl check first.
- **vfs_is_dir:** Used `ops->readdir != NULL` as a fallback for directory detection. All tmpfs nodes share the same ops table, so every file was reported as a directory.
- **Spurious Enter from autorepeat:** The AR state was not reset on Enter, causing repeated Enter presses after any command that took more than 250ms.

## Roadmap (Phase 13+)

- ELF loader: `process_execve()`, `load_elf()`, initial stack with `argc/argv/envp/auxv`
- Dispatcher syscall: MSR_LSTAR + assembly handler + dispatch table
- Priority syscalls: `execve`, `rt_sigaction`, `setsid`, `getdents64`, `nanosleep`
- COW page fault handler
- Libc completion
- Userspace: init, shell, coreutils (sources already present, waiting for the ELF loader)

---

> **Fun fact**: This text was also generated by the AI ​​(Claude Sonnet 4.6 Extended).

> **Note**:
> - The command output is in Italian.
> - The actual version of the project is 5.7, considering all development sessions. It is published on GitHub as v1.0.
> - The entire project is documented in detail in the `OS_PROJECT_BLUEPRINT.md` file, generated by the AI ​​during development. It is included in the ZIP with the source files.
> - An English translation is planned.
