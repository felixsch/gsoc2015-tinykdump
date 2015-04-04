#Prototypes

Sources listed here are only prototypes and __not__ fully functional code. Patches currently only tested with kernel version _3.18.8_ and _x86_64_.

###Kexec CMA patch

kernel options required:

    CONFIG_KEXEC=y
    CONFIG_KEXEC_USE_CMA=y
    CONFIG_CMA=y
    CONFIG_CMA_DEBUG=y
    
This patch enables kexec to allocate memory from a cma area allocated at startup.

By adding `crashkernel=sizeM` to kernel commandline the kexec cma area is initialized.

Get available size:

    cat /sys/kernel/kexec_crash_size
    
To allocate cma memory:

    echo <size> > /sys/kernel/kexec_crash_size
    


###Pstore patch

By patching pstore, console logging is enabled optionally.

kernel options required:

    CONFIG_PSTORE=y
    CONFIG_PSTORE_CONSOLE=y
    
to enable the console add `pstore.console=1` to the kernel commandline.


###tinykdump python service

coming soon.

    



