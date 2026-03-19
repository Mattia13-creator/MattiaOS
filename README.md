# MattiaOS: Un Sistema Operativo Generato dall'IA (Italian version. For the English version, scroll down.)

Questo progetto è un sistema operativo completamente funzionante, la cui architettura e base di codice sono state generate interamente tramite **Intelligenza Artificiale**. Sviluppato in Italia come proof-of-concept, dimostra le straordinarie capacità di programmazione dei moderni modelli linguistici di grandi dimensioni, in questo caso Claude Sonnet 4.6 Extended.

Il risultato è un sistema operativo creato da un ragazzo di 14 anni e una AI, senza che una singola riga di codice fosse scritta manualmente da un autore umano.



## 🧠 Metodologia di Sviluppo
L'intero processo di sviluppo — dal design del kernel fino alla generazione dell'immagine ISO avviabile — è stato gestito delegando i task all'IA.

*   **Compilazione In-Chat:** I compilatori necessari sono stati forniti all'IA direttamente tramite l'interfaccia di chat, permettendole di compilare il codice sorgente in binari e assemblare l'immagine ISO in autonomia.
*   **Toolchain:** Il sistema è stato compilato utilizzando **NASM** per i componenti Assembly e un **GCC Cross-Compiler** per il codice C, garantendo l'assenza totale di dipendenze da librerie di sistema esterne.

## ⚙️ Architettura Tecnica
Il sistema utilizza lo standard **El Torito** per il boot, carica un kernel in formato **ELF** e inizializza l'hardware passando alla modalità a 64 bit (**Long Mode**).

*   **Kernel e Syscall Personalizzati:** Sebbene il design del filesystem tragga ispirazione da Linux, il kernel e le chiamate di sistema (syscall) sono stati scritti da zero e sono interamente proprietari.
*   **Ecosistema Standalone:** A causa della sua architettura syscall personalizzata, l'IA ha sviluppato un set di comandi core su misura per consentire l'interazione diretta con la shell di sistema.

## 🛠️ Comandi Disponibili
La shell di sistema supporta attualmente i seguenti comandi nativi:

| Comando | Descrizione |
| :--- | :--- |
| `help` | Elenca e descrive tutti i comandi disponibili |
| `ver` | Mostra la versione attuale del sistema operativo |
| `mem` | Mostra le statistiche della RAM e i processi attivi |
| `pci` | Elenca i dispositivi PCI e le periferiche connesse |
| `ls` | Elenca i file e le directory nel percorso corrente |
| `cd` | Cambia la directory di lavoro corrente |
| `open` | Visualizza il contenuto dei file in formato esadecimale |
| `edit` | Editor esadecimale integrato per la modifica dei file |
| `tree` | Stampa la struttura completa dell'albero del filesystem |
| `clear` | Pulisce la schermata del terminale |
| `halt` | Esegue lo spegnimento sicuro del sistema |
| `diag` | Esegue la diagnostica interna del sistema |

## 🐛 Bug Noti e Limitazioni
In quanto progetto sperimentale, il sistema presenta alcune instabilità:
*   Inesattezze occasionali durante l'apertura di specifici tipi di file.
*   Problemi di rendering all'interno dell'albero delle directory (`tree`) in percorsi profondamente annidati.


> **Curiosità**: Anche questo testo è stato generato dall'IA (Gemini 3 Flash).

> **Note**:
* L'output dei comandi è in italiano, ma viene fornita una traduzione in inglese.
* La versione reale del progetto è la 5.7, considerando tutte le modifiche. Su Github è la 1.0.
* Tutto il progetto è documentato meglio nel file "OS_PROJECT_BLUEPRINT.md", perché è stato generato dall'AI durante la creazione. Si trova nello zip con i file sorgenti dentro.


---


# MattiaOS: An AI-Generated Operating System (English Version)

This project is a fully functional operating system whose architecture and codebase were generated entirely by **Artificial Intelligence**. Developed in Italy as a proof-of-concept, it demonstrates the extraordinary programming capabilities of modern Large Language Models, in this case Claude Sonnet 4.6 Extended.

The result is an operating system created by a 14-year-old boy and an AI, without a single line of code being written manually by a human author.



## 🧠 Development Methodology
The entire development process—from kernel design to the generation of the bootable ISO image—was managed by delegating tasks to the AI.

*   **In-Chat Compilation:** The necessary compilers were provided to the AI directly through the chat interface, allowing it to compile source code into binaries and assemble the ISO image autonomously.
*   **Toolchain:** The system was compiled using **NASM** for the Assembly components and a **GCC Cross-Compiler** for the C code, ensuring a total absence of external system library dependencies.

## ⚙️ Technical Architecture
The system utilizes the **El Torito** standard for booting, loads an **ELF** format kernel, and initializes hardware by transitioning into 64-bit mode (**Long Mode**).

*   **Custom Kernel & Syscalls:** While the filesystem design takes inspiration from Linux, the kernel and system calls (syscalls) were written from scratch and are entirely proprietary.
*   **Standalone Ecosystem:** Because of its custom syscall architecture, the AI developed a bespoke set of core commands to enable direct interaction with the system shell.

## 🛠️ Available Commands
The system shell currently supports the following native commands:

| Command | Description |
| :--- | :--- |
| `help` | Lists and describes all available commands |
| `ver` | Displays the current operating system version |
| `mem` | Shows RAM statistics and active processes |
| `pci` | Lists connected PCI devices and peripherals |
| `ls` | Lists files and directories in the current path |
| `cd` | Changes the current working directory |
| `open` | Displays file contents in hexadecimal format |
| `edit` | Integrated hex editor for file modification |
| `tree` | Prints the complete filesystem tree structure |
| `clear` | Clears the terminal screen |
| `halt` | Executes a safe system shutdown |
| `diag` | Runs internal system diagnostics |

## 🐛 Known Bugs & Limitations
As an experimental project, the system exhibits some instabilities:
*   Occasional inaccuracies when opening specific file types.
*   Rendering issues within the directory tree (`tree`) in deeply nested paths.


> **Curiosity**: This text was also generated by AI (Gemini 3.2).

> **Note**: 
* the output of the commands is in Italian, but an English translation is provided.
* The real version of the project is 5.7, considering all the changes. On Github is 1.0.
* The entire project is better documented in the "OS_PROJECT_BLUEPRINT.md" file, because it was generated by the AI ​​during creation. It's in the zip file with the source files.
