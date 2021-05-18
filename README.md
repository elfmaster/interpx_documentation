# InterpX Runtime Description

InterpX is an ELF runtime technology. This technology makes possible the
loading and execution of modules within the userspace process memory, and is
designed for the purpose of creating a wide variety of things that require
program transformation and runtime instrumentation, such as arbitrary runtime
microcode patching, security plugins, bug fixes, and code optimization.

## How InterpX works

ELF Binaries which are dynamically linked have a PT_INTERP segment which
contains the dynamic linker path, i.e. "/lib64/ld-linux.so". This path gets
replaced with the path to "/usr/local/interpx/bin/interpx". In statically linked
binaries, we insert a PT_INTERP segment with this path.  The custom program
interpreter which we call "interpX" is loaded by the kernel at program runtime.
The InterpX interpreter is loaded in place of the dynamic linker, and is
responsible for setting up the InterpX runtime environment, loading and
executing custom modules, very similar to loadable kernel modules; we call them
"Loadable process modules". Once modules have been loaded, and various hooks and
probes are in-place, the InterpX interpreter passes control to the dynamic linker.
The dynamic linker performs its magic, and then passes control to the target
executable. The modules functions will be executed at various phases of execution
depending on their declaration type, as we will describe shortly.
```
--Flow of execution on a dynamically linked executable

[Kernel ELF loader]
 	|
	InterpX -> ld-linux.so ->  target_elf_executable
	|	|	        
(Loads modules) (modules may be invoked at any point)
	\	      \
	 module1.o    module2.o

--Flow of execution on a statically linked executable

[Kernel ELF loader]
	|
	InterpX -> target_elf_executable
	|
	(Loads modules)
	\	      \
	module1.o     module2.o
```
Modules are written in C, and loaded into the process image as relocatable
objects.  The InterpX-runtime creates a private heap, data segment, and .bss
for these relocatable objects suffixed with a ".xo" file extension. The
InterpX-runtime offers a rich API for easy access into the process data
structures, such as the auxiliary vector, the .got.plt, the VDSO, etc. of the
running process. The InterpX API is not yet designed, but will offer clean
programmability to developers who want to write modules for
microcode-patching-systems, security plugins, or anything else that requires
heavy program transformation or memory instrumentation. The Linux process
images are manifold in structural nuances and require great depth of insight to
instrument properly. The InterpX runtime environment aims to make this a
standard part of the ELF runtime ABI, in a way that is straightforward,
performant, and secure. Programmers should find it as a friendly way to create
complex systems for hotpatching, or for designing security plugins such as
sandboxes and exploitation mitigation systems.

Modules contain functions that can be executed within four distinct phases of
the Linux process runtime:

- PRE-LDSO:	Invoked prior to the dynamic linker
- POST-LDSO:	Invoked after the dynamic linker
- IN-EXE:	Invoked from within an executable hook (i.e. .got.plt, function trampoline, probe, etc)
- POST-EXE:	Invoked after the executable runs

## eBPF probes

In the future I plan to incorporate eBPF capabilities into InterpX runtime, as it will
give us kprobe capabilities for implementing some kernel hooking capabilities
into our code.

## A major PRO about InterpX vs. PTRACE

Currently most process image instrumentation requires the ptrace(2) system call, in order
to attach to the process, stop it, apply the patch, and then resume the process. This is
extremely slow, especially when the need to patch mission critical software is imperative.

## Some use-cases for InterpX

### A debugger specifically for debugging the dynamic linker.

This is a very singular and niche use-case, but a great example nonetheless
of how InterpX can accomplish novel tasks.
Certain aspects of the dynamic linker are very difficult to debug because it
runs prior to any program, even the debugger itself. InterpX allows modules to
execute prior to the dynamic linker, which allows for the setting of
breakpoints, and thread creation for a child process that could serve as the
monitor thread for following execution and catching signals (breakpoints) from
the dynamic linker.  InterpX also allows for loaded modules to be invoked from
hooks within the target program itself, therefore functions within the dynamic
linker can be easily hooked with a handler function that gives in-depth debugging
insight. InterpX would be a good platform for designing any type of userland
debugger really.

## An advanced microcode patching system for runtime

An entire microcode patching system can be implemented LPM's (Loadable process
modules).  Very similarly to how the kernel can load LKM's (Loadable kernel
modules) for implementing everything from software drivers to completely new
kernel features, plugins, and patches.

Imagine a microcode patching system used for introducing new patches into a
code-base without requiring its source code (A problem with some legacy
software). Let me give an example of how this might work with two modes of
operation; Patches are applied at arbitrary runtime (Hotpatching), or at the
time a process restarts in which time it is patched into memory at the
beginning of runtime.

Example: A mission critical program for monitoring SCADA and ICS power-control has been recently
found to have a security vulnerability in it, and must be patched.

1. /usr/sbin/critical_ctl
This software must keep running at all times, and therefore we will write a
patch designed to be loaded and applied in the middle of this execution. The
function 'foo' that needs to be patched is small, and requires a change in integer
type on the stack, in order to prevent an integer overflow vulnerability.

2. We discover that critical_ctl does not have any dwarf debugging data, which
limits the granularity at which we can patch, so we decide to implement a function
hook which invokes our replacement version of the function that uses the correct
integer type to fix the vulnerability.

3. We write a replacement version of the function and compile it with the interpX
gcc frontend. 
```
/*
 * foo_patch.c
 */
#include "interpx.h"
#include "command-ctl.h" /* For struct somedata */

__IPX_COMM_NAME "command-ctl" /* Name of processes to patch */
__IPX_PATCH_TYPE IPX_TYPE_HOTPATCH /* Hotpatch all instances of "command-ctl" */

int __attribute__((__IPX_HOOK, LOCAL_BINDING))
foo(void *data, uint8_t *inbuf)
{
	struct somedata *ptr = (struct somedata *)data;
	/*
	 * The original foo() function used 'int len'
	 * here, and we've created a replacement foo()
	 * which uses the correct integer to prevent
	 * an integer underflow.
	 /
	ssize_t len = data->p_len;
	int ret;

	if (len < PAGE_SIZE) {
		ret = copy_single_page(data, inbuf, len);
	} else {
		ret = copy_all_pages(data, inbuf);
	}
	return ret;
}
```

4. Compile this into a relocatable object via ipx-gcc -c foo_patch.c -o foo_patch.xo
5. Place patch.xo in /usr/share/interpX/patches/foo_patch.xo
6. The InterpX runtime has an inotify signal handler set to watch for new patches
7. The InterpX runtime applies a thread-safe and SMP safe (Since text pages are shared accross processes)
function trampoline by modifying the call offset's to function 'foo()'.
8. All future calls to foo() are directed to our secure version of foo.

All of the patch files to be applied to processes are placed within
/usr/share/interpX/patches.  Depending on whether or not the patch type is
"PRE_PATCH" or "HOTPATCH" it wil either be applied at the beginning phase of
execution (Prior to the executables entry point), or within the middle of
execution (Hotpatch).

## Security modules

Security modules for exploitation mitigation systems, sandboxes for hardening,
and overall process image hardening against APT's (Advanced persistent
threats). The idea of implementing security features to harden the process
image was my original motivation for beginning the design of InterpX. 

## Summary

In short, InterpX can be used to load runtime modules who's functionality can
expand the logic and capabilities of a running process, by arbitrary
modifications to its runtime environment.

ElfMaster -- Ryan@bitlackeys.org

