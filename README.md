<br/>
<br/>
<p align="center">Tinykdump</p>
<p align="center">Improving kernel crash analysis</p>

<br/>
<p align="center">March 17, 2015</p>

###Summary

In every software development the error analysis is one of most crucial parts when resolving issues. In the current Linux kernel kdump[1] is used to provide system logs and memory dumps immediately after the system crashes. This project's intention is to minimize issues within the kdump infrastructure and make use of the features of the modern Linux kernel.

###The project

Advanced kernel debugging relies on a mechanism to analyze the root cause of a kernel crash. Kdump provides a memory dump and kernel buffer of the crashed kernel. The current implementation uses kexec to load a second kernel, the crashkernel, which is executed when the first kernel crashes, to capture the desired information. To store the crashkernel, kexec reserves a fixed amount of physical memory which is available during kernel's start. Currently every distribution ship there own tools to manage loading the crashkernel image. Fedoras implementation uses dracut to prepare the crashkernel image. The generated crashdump can be stored in several ways (harddisk, nfs-storage, usb, via ssh).

The goal of the project is to resolve current issues and provide a more generic reliable way of retrieving kernel crash dumps. Furthermore, it will coexists with the current kdump implementation. In order to remove the restriction of reserving the crashkernel memory at start up which renders the memory useless until a crashkernel is loaded, the kernel-side code of kexec will be improved by using the new re-factored contiguous memory allocator (CMA)[2]. This makes the reserved memory reusable to movable pages until the crashkernel is loaded.

Current crashkernel debugging is tedious because of the lack of a reliable way to make the kernel mode settings (KMS) cooperate with the kdump crashkernel, as this almost always results in no usable display output.
To provide a way of debugging the crashkernel, pstore will be utilized to store the crashkernels console output.

Fedoras current implementation of the kdump service is powerful and offers several features. All the features comes with the price someone maintaining and testing the implementation. This might work in a static environment where the kernel rarely changes but Fedora is based on fast-paced development and therefore the changes happen frequently. Tinykdump will reduce features to gain more maintainability. The current kdump service utilizes dracut[8] to generate the kernel images. To overcome the dependency on dracut, the goal is to write a service which generates kernel images by using the busybox-toolchain, usb drivers and makedumpfile. Therefore, instead of supporting several ways of storing the crashdump , tinykdump will only support external usb drives.

###Benefits for Fedora and open-source community

   * By implementing a generic service, with a minimized feature set and no dependencies to any initramfs generators, maintaining efforts can be reduced. Likewise porting the service to different platforms is made less troublesome.

   * The goal of tinykdump is to provide a reliable and uncomplicated mechanism to debug the kernel, hence debugging kernel issues in Fedora can be more easy.
   
   * Enable pstore to store the crashkernel's console ouput makes it more easy to discover potential issues in current crashkernel configurations.

   * Memory reusability is further maximized, by providing patches to enable CMA support to kexec. In addition, low-memory systems can be supported due the reused memory and small crashkernel, generated by the user-space service. 

   * Patches for CMA support and pstore are usable not only to tinykdump, but also to kdump.
   

###Implementation details

* #####kexec dynamic memory reservation
Instead of reserving the memory at kernel start up, a new CMA area is initialized. By using a wrapper function `crash_init_reserve_memory(...)` which calls `cma_init_reserve_mem(..)` architecture depended code can be injected. Until now the currently used `struct resource crashk_res` is still initialized to default values, to do not break compatibility with kexec-tools. On write to `/sys/kernel/kexec_crash_size` memory allocation is triggered and `crash_contigious_request(..)` is called. `crash_contigious_request(..)` uses `cma_alloc` to move used pages aside. In addition, `crashk_res` is updated to the allocated space from `cma_alloc` and inserted into `iomem_resources`. By using a new configuration flag `CONFIG_KEXEC_CMA` this feature can be optional enabled.


* #####Using pstore to log console crashkernel output
Pstore uses a generic storage system where new pstore types can easily added. To add support for kexec, `PSTORE_TYPE_KEXEC_CONSOLE` is added to `enum pstore_type_id`. To capture the actual console output a new console is created and registered to the system via `register_console` which uses `psinfo` to write to available pstore. (how to determine if we are in a crashkernel ???)

* #####Implementing a python based tinykdump service
The new python services will be based on the current kdump implementation of Fedora[9]. The `configparser` module of the standard python library is used to parse configuration which is located in `/etc/default/tinykdump`. The initramfs which captures the dump and stores them onto usb drive uses a static linked busybox (which is available via Fedora packages) and makedumpfile from kexec-tools.
Initramfs generation is done the manual way, generating the directory structure in `/var/run/tinykdump` and compress it using cpio. The current installed kernel is used as crashkernel. If no reserved memory is found the service tries to write the size which is required, determined by the size of the crashkernel image generated earlier, to `/sys/kernel/kexec_crash_size`. If CMA support is enabled the memory is reserved, otherwise failure. If memory is already reserved and the size is less than required. Try to shrink the reserved memory.
Functions which are supported by the python service are:
```
    -c | --c <config> use custom config location
    -b | --build build/update initramfs image
    -m | --module <module, module> additional modules which should be loaded by initramfs
    -l | --load load crashkernel image. (Implicit build if not done yet)
    -s | --script <bash file> script which should executed before the dump is written
```


###Timetable

* #####Now — 2 April (pre-selection phase)
Looking into the kernel sources to fully grasp the concept of CMA and pstore. Working on proof-of-concept patches for dynamic memory reservation and crashkernel pstore integration. Discussing how real implementations should possible be like. _(Vacation from 2 till 6 April)_

* #####6 April — 27 April (pre-selection phase)
Implement a proof-of-concept python service which allows basic kexec loading (without secure boot). Learning about how secure boot/ secure mode works and working on a testing/debugging strategy. Discuss with my mentor about the concrete implementation of the service.


* #####27 April - 6 May (pre-coding phase)
* Write the real CMA kernel patch _(Vacation from 6 till 11 May)_

* #####11 May — 24 May (pre-coding phase)
Testing the CMA patch implementation and debugging/fixing potential issues. Getting the patch community reviewed. Writing a blogpost about my progress, the current status of my work and my new insights.

* #####25 May — 31 May (first coding phase)
Implementing the real pstore kernel patch

* #####31 May — 14 June (first coding phase)
Testing the pstore patch and debugging/fixing potential issues. Getting the patch community reviewed. Applying further changes to the dynamic memory reservation patches if requested. Writing a blog post about what I have learned about pstore and what I have done so far.

* #####14 June — 26 June (first coding phase)
Building kernel images for x86/x86\_64 with both patches applied and build them. Running tests on virtual systems and hardware which are available to me. Fixing issues which are found while testing. Writing a blogpost about how to test kernel patches properly. __Optional__: Building the images on a RHEL base and testing them.

* ###26 June Midterm evaluation

* #####26 June — 14 July (second coding phase)
Implementing the python service with features described in "implementation details".

* #####14 July — 1 August (second coding phase)
Testing the service and fixing potential issues. Testing on virtual systems and hardware which is available to me. Write about the current status of my work, on my blog.

* #####1 August - 21 August (second coding phase)
More testing and bug fixing. If further changes to the patches have to be done, they should be done now. Writing documentation about the service. Create manpages etc.

* #####Future
After google code of summer I will port tinykdump to other distributions like openSUSE and debian. If not already done: Port the dynamic reservation patches to other platforms if possible.
If desired I'm happy to maintain the tinykdump package in fedora. 


###About me

Name: Felix Schnizlein<br/>
E-Mail: felix@none.io<br/>
IRC: felixsch (in freenode and oftc)<br/>
Wiki page: [https://fedoraproject.org/wiki/GSOC_2015/Student_Application_felixsch](https://fedoraproject.org/wiki/GSOC_2015/Student_Application_felixsch)



