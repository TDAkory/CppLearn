# A General Understanding when talk about 'C/C++ binary start up'

- [A General Understanding when talk about 'C/C++ binary start up'](#a-general-understanding-when-talk-about-cc-binary-start-up)
  - [How to understand "C runtime"?](#how-to-understand-c-runtime)
  - [ELF entry-point](#elf-entry-point)
  - [Ref](#ref)

## How to understand "C runtime"?

C programs don’t run “on top” of any runtime in the way that Java/python/JS/etc programs do, so usually when you hear the term “C runtime,” it’s just a poor piece of terminology for the startup routines that get automatically linked into your program by the compiler (i.e. the code that calls main() and initializes global variables). These routines are shipped as part of the compiler and reside in the crt0.o object file, usually. They implement (on Linux and in most bare-metal ELF programs) a function called _start, which contains the very first code your program runs when it is exec’d by the OS (or the firmware’s bootstrap code, in the case of bare-metal). On hosted platforms (i.e, ones with an OS), the crt0 is also responsible for initializing the C standard library—things like malloc(), printf(), etc.

It’s possible to specify to gcc or clang an alternate crt0 object file, or to exclude one altogether, in which case you’d need to define your own _start() function in order for the program to be linked into a working executable.

C++ uses something similar, but with much more complexity in order to support exceptions and constructors/destructors.

Nevertheless, once your program has been compiled, this “extra” code is no different from the perspective of the OS/CPU than any other code you’ve linked to in your program.

## ELF entry-point

> [ELF Header](https://docs.oracle.com/cd/E19120-01/open.solaris/819-0690/chapter6-43405/index.html)

```c
#define EI_NIDENT       16
 
typedef struct {
        unsigned char   e_ident[EI_NIDENT]; 
        Elf32_Half      e_type;
        Elf32_Half      e_machine;
        Elf32_Word      e_version;
        Elf32_Addr      e_entry;
        Elf32_Off       e_phoff;
        Elf32_Off       e_shoff;
        Elf32_Word      e_flags;
        Elf32_Half      e_ehsize;
        Elf32_Half      e_phentsize;
        Elf32_Half      e_phnum;
        Elf32_Half      e_shentsize;
        Elf32_Half      e_shnum;
        Elf32_Half      e_shstrndx;
} Elf32_Ehdr;

typedef struct {
        unsigned char   e_ident[EI_NIDENT]; 
        Elf64_Half      e_type;         // Identifies the object file type
        Elf64_Half      e_machine;      // Specifies the required architecture for an individual file.
        Elf64_Word      e_version;
        Elf64_Addr      e_entry;        // The virtual address to which the system first transfers control, thus starting the process. If the file has no associated entry point, this member holds zero
        Elf64_Off       e_phoff;
        Elf64_Off       e_shoff;
        Elf64_Word      e_flags;
        Elf64_Half      e_ehsize;
        Elf64_Half      e_phentsize;
        Elf64_Half      e_phnum;
        Elf64_Half      e_shentsize;
        Elf64_Half      e_shnum;
        Elf64_Half      e_shstrndx;
} Elf64_Ehdr;
```

## Ref

- [Runtime library](https://en.wikipedia.org/wiki/Runtime_library)
- [C and C++ runtime libraries](https://developer.arm.com/documentation/100073/0620/The-Arm-C-and-C---Libraries/C-and-C---runtime-libraries)
- []()