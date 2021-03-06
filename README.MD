## What's the MemITM tool? ##

The MemITM (Mem In The Middle) tool has been developped in order to easily intercept "messages" in Windows processes memory. We developped a lot of custom memory interception tools in order to capture network messages before encryption, or IPC messages, and to be able to inspect them or alter them to do some fuzzing. Each tool was really custom, not generic, implemented in C/ASM and was not easy to use/maintain/adapt.

The MemITM tool has been developped in order to address these problems, and consists in :
- an IDA Python script, which generates a "config" file, indicating where and how the interception points have to be placed (relative address, how to find the buffer and its size, how to place the hook, etc.) ;
- the DLL file, which will be injected in the target process and will loads the config and places the hooks ;
- a DLL file injector, which will load the DLL in the target process ;
- a python script, which communicates with the injected DLL/hooks, and gets in realtime the intercepted buffers (and can alter them) in really simple callbacks you can modify as you want.

You may download it (sources + compiled) here: https://github.com/Amossys/MemITM

## How simple is MemITM? (Example1: WriteFile) ##

Let's say you want to intercept file writes (there are simpler ways to do it, but that's for the example) in a specific Windows process. File writes are performed by the `WriteFile` function, which is exported by `kernelbase.dll`, and follows the scheme `BOOL WriteFile( HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, ...)`. WriteFile is a `__fastcall` function : `lpBuffer` is pointed by `RDX` and `NumberOfBytesToWrite` by `R8`. Let's intercept these buffers.

1. open your "kernelbase.dll" file in IDA Pro, load the `generate.idapython.py` script, and just run `getHook(LocByName("WriteFile"), "rdx","r8")` followed by `updateConfig("config.bin")`. 

2. run a notepad and then, `python memitm.py notepad.exe config.bin`.

That's it! A hook has been placed on the `WriteFile` function, and the `memitm.py` "logger" and "fuzzer" functions will receive a copy of the buffer.

## My buffer is not pointed by a register! (Example2: NtCreateFile) ##

The `getHook` function allows specifying registers for the buffer and its size, but in some cases it may not be the case. Let's say you want to intercept `NtCreateFile` syscalls and get the filename. `NtCreateFile` follows the scheme `NTSTATUS NtCreateFile( OUT PHANDLE FileHandle, IN ACCESS_MASK DesiredAccess, IN POBJECT_ATTRIBUTES ObjectAttributes, ...)`, and the file name can be obtained by `ObjectAttributes->ObjectName->Buffer` (size is `ObjectAttributes->ObjectName->Length * sizeof(WCHAR)`).

`getHook` also allows specifying a shellcode in its `t1customOpcodes` parameter instead of registers. This shellcode must :
- place the buffer pointer into the `RCX` register;
- place the buffer's size into the `RDX` register;
- not mess up with the stack.

In the `NtCreateFile` case, we can just run this ASM code to do the work:

    mov rax, [r8+0x10]    ; rax is now ObjectAttributes->ObjectName
    mov rcx, [rax + 0x10] ; rcx is now ObjectAttributes->ObjectName->Buffer
    mov rdx, [rax]        ; rdx is now ObjectAttributes->ObjectName->Length
    and rdx, 0xFFFF       ; Length is a USHORT 
    shl rdx, 1            ; and must be *2

Let's assemble this (using for instance the online disassembly website) : `498B4010488B4810488B104881E2FFFF000048D1E2`. Our `getHook` call should now be `getHook(LocByName("NtCreateFile", t1customOpcodes="498B…E2")`.

## Can I intercept multiple calls? ##

Yes! The config file can embed multiple interception points definitions, and the `getHook` function appends the file. You can set interception points in multiple modules. For instance, you can intercept ALPC and IOCTL messages at the same time.

You will be able to diffentiate them by their "message ID" in the python callbacks.

## Oops, BSOD/system freeze! ##

Don't worry, there is also a simple HTTP server (`logserver.py`) and the `httpNetSend` function which allow you to send your test cases (and modification) to a remote host in realtime.

## How do I fuzz? ##

Well, you can start using the `bufferBitFlip` function in order to... flip several bits. Logging raw messages will allow you to write your own dissector, and to start implementing manually a better fuzzer :). Remember: you can't change the buffer size!

For instance, in our `WriteFile` example, the following `fuzzer` function will replace "hello" by "world":

    def fuzzer(data, msgID=None, pid = 0):
       return data.replace("hello","world")

## Example3: dissecting ALPC messages ##

For instance, for ALPC messages, you may have the following `memitm.py` setup:

    def logger(data, msgID=None, pid=0):
        totalLen = 0
        dataLen = 0
        typ = 0
        dataInfoOffset = 0
        cid = 0
        tid = 0
        messageID = 0
        clientViewSize = 0
        # skip the PORT_MESSAGE header (0x18 bytes)
        if len(data) > 0x18:
            totalLen = struct.unpack("<H",data[:2])[0]
            dataLen = struct.unpack("<H",data[2:4])[0]
            typ = struct.unpack("<H",data[4:6])[0]
            dataInfoOffset = struct.unpack("<H",data[6:8])[0]
            zeroInit = struct.unpack("<L",data[4:8])[0]
            cid = struct.unpack("<L",data[8:0xC])[0]
            tid = struct.unpack("<L",data[0xC:0x10])[0]
            messageID = struct.unpack("<L",data[0x10:0x14])[0]
            clientViewSize = struct.unpack("<L",data[0x14:0x18])[0]
        logMessage = "ALPC message : %x:%x - %x:%x - %d:%d - %x - %x" % (totalLen, dataLen, typ, dataInfoOffset, cid, tid, messageID, clientViewSize)
        print logMessage
        return

## How does it works internally? ##

The IDA Pro script embeds a simple "hooking" engine, which allows moving several instructions to a specific area (derived from our *DIMCT* tool). It must handle lots of corner cases, such as memory relative instructions, cross references, etc. and will probably display lots of "can't install hook here" if you try intercepting really small basic blocks (<5 bytes for x86 binaries, <12 bytes for x64 ones) which contain multiple cross-references. For more information, just read the source code :).

It generates the following information in the config file:
- the module name;
- the relative address of the interception point;
- the interception hook area (aka "T1 trampoline"), which places the buffer/length into `RCX`/`RDX` and calls the DLL log routine;
- the restoration hook area (aka "T2 trampoline"), which restores the context and executes replaced instructions before returning to the interception point;
- the restoration hook area relocations, for relative instructions which have been converted to absolute ones and must be patched.

The DLL injector is a simple DLL injector (`CreateRemoteThread`/`LoadLibraryA` stuff).

The DLL itself initiates a shared memory area and waits for the config data (the shared memory area has 3 generic fields : the shared memory message ID, the message size and the message buffer). Once received, it parses and installs the hooks in memory, and no config update will be allowed after this (you must kill the process if you want to set up new hooks). Any placed hook will land in the `genericHookFunction` DLL function. This function just fills the memory area (protected with critical sections) with the buffer data, its size and the message ID (set in higher bits of the shared memory message ID field). In order to notify the python process, 2 events are used to signal new messages and to wait for the process to patch them (timeout is 2 seconds). If the buffer has been updated, the original buffer is modified too.

The `memitm.py` script runs the injector process, and waits for the shared memory to be created. It then sends the configuration and waits for the message event. When received, the shared memory is read and the `logger` and `fuzzer` functions are called with the buffer, the process ID and the message ID. If the `logger` function returns a different buffer than the original one and its size is equal, it is written to the shared memory area. The "ACK" event is then signaled, and the script waits for a new message.

## That's it! ##

We hope this tool will be useful, any contrib/reviews will be appreciated! Also, if you're French and like Rennes, we're hiring :)

