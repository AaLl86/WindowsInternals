## Windows Internals Foundamentals training
This repro contains all the material for the Windows Internals training which is distributed through the O'Reilly learning platform:

https://www.oreilly.com/live-events/windows-internals-fundamentals/0636920095044/

### Step 1 - Download the needed material
For the training experiments we will use a Virtual machine with Windows 11 installed connected with a local and remote kernel debugger.

Here are the links with the needed software:
1.  Windows 11 ISO: https://www.microsoft.com/software-download/windows11
1. Windows SDK (includes the Windbg debugger): https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/
1. The latest SkTool version is included in this repository
1. HyperV is included in Windows 11.
1. Putty: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
1. PsTools: https://download.sysinternals.com/files/PSTools.zip


### Step 2 - Set up the host environment
The first thing that you need to do is to install the Debuggers and SkTool. You can do this by downloading the Windows SDK and by executing the Setup file. After you have finished you should have the Windbg debugger located in `C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe`.

To install SkTool, just copy the correct SkTool version from this repository in your local workstation and Virtual machine that we are going to create.

#### Create the VM
Your system should have already installed the HyperV role. If, from the start menu, you do not see any "Hyper-V Manager", it means that the feature is not installed. To install it just open the Control Panel, and click on "Program and Features". On the top left corner of the window there should be a link named "Turn Windows Features on or off". Click on the link, scroll the Windows features list until you see "Hyper-V". Fully select the feature and click OK. Windows should correctly install Hyper-V hypervisor for you.

Open the Start menu and search "Hyper-V Manager". Right-click on your workstation name (under "Hyper-V Manager" in the top left corner) and select "New -> Virtual Machine...". Click Next, specify a name and select "Generation 2" (important for this training), Assign then at least 4 GB of physical memory (disable Dynamic memory), since the Windows 11 setup program checks this prerequisite. Configure the Network and the Virtual disk (minimum 128 GB). At the "Installation Options" phase make sure to select "Install an operating system later". Complete the Wizard by selecting the Finish button.

Now on the center of your Hyper-V Manager window, the new VM should appear. Right click on it and select "Settings...":
1. From the "Security" tab make sure that the "Enable Secure Boot" option is **not** selected. Mark the "Enable Trusted Platform Module" though. This is needed for Windows Setup to work.
1. From the "Processor" tab make sure to assign to the VM at least 2 Virtual CPUs (otherwise it would be very slow)
1. From the "SCSI Controller" tab click on "DVD Drive" and then on the "Add" button. This will create another choice under "SCSI Controller": select the "DVD Drive" which will show another tab.
1. From the "DVD Drive" tab select "Image file:" and browse to the location when you stored the Windows 11 ISO
1. Return back to the "Firmware" tab and make sure that the "DVD Drive" is the first boot entry in the order.
1. Click OK and close the Settings window.

Your VM is now ready to start. Double click on it and click "Start". When the VM shows "Press any key to boot from CD or DVD..." make sure to press any key to let the Windows Setup process start. Finish the Windows Setup process and wait that Windows is correctly started in the guest VM...

#### Set up the VM 
Setting up the VM is relatively simple. First, you need to make sure that the host and the guest can communicate correctly



md c:\windows\symbols

srv\*C:\windows\symbols*https://msdl.microsoft.com/download/symbols


```
Experiment Segment 2 - Witness hardware interrupt
!idt
Take the KINTERRUPT from the PRCB when generating the Interrupt frame
* Read IA32_GS_BASE (careful, not IA32_KERNEL_GS_BASE) to get the PCR
rdmsr 0xc0000101
dx ((nt!_KPCR*)Pcr)->Prcb->InterruptObject[Vector]


Experiment segment 3 - Peeking in the process and thread data structures
* Dump the putty processes
!process 0 0 putty.exe
* Go deeper (without stack)
!process <Proc> 2
* Now analyze the data structures
dt _EPROCESS <Proc>
* Note the difference between the PCB (part of the Executive Process) and the PEB
dt _EPROCESS <Proc> Peb
* You can not read the PEB in user memory, since you are attached to another process
.process
* You should switch the context to the correct process
r cr3
.process /p /i <Proc>
g
.process
r cr3
* Now you can dump the PEB (check especially the process parameters)
dt _EPROCESS <Proc> Peb


Experiment Segment 4 - Viewing a PTE and understanding CA
* First let's put a BP on the function that Windows uses to map an image section
bp nt!MiMapViewOfImageSection
g
* When the debugger break we know that an application or the system is
* mapping a section.
* MiMapViewOfImageSection(PCONTROL_AREA Ca, PVOID Params, PVOID *BaseAddress, ...)
* Let's dump the CA and the base address pointer and resume the execution until 
* the function finishes
r rcx,r8
g @$ra

* Let's analyze the leaf PTE now of the base VA (resolved into old R8 register)
!pte <Base VA>
* The leaf PTE is invalid, since, as explained in the training a VAD exists
* Let's observe the PTE transitions by placing an hardware BP on the leaf page
bc *
ba w8 <LeafPte>
g
* The leaf PTE goes from Invalid to Software Prototype one (linked to the Proto PTE)
!pte <Base VA>
g
* Now the leaf PTE goes from Software to valid (is materialized)
* Now let's go and check the CA and the proto PTEs:
!ca <Control Area>

* As you can see each Subsection has its own contiguous Proto PTEs, which can be
* even them in Software format
dq <ProtoPtes> l4

* you can repeat the experiment and print out the stack
* Finally, let's check the valid PFN into the leaf PTE:
!pfn <Leaf_Pfn>


Experiment Segment 5 - Exploring the Virtualization status of your system
* Just run SkTool from a system where HyperV is enabled
* For seeing the anatomy of a Trustlet repeat Experiment 3 with the "lsaiso.exe" process
!process 0 0 lsaiso.exe
!process <LsaIso proc> 

* You will notice that the extension is not able to show any user-stack since 
* they are all in the secure world. (you will see only PspSecureThreadStartup 
* which calls VslpEnterIumSecureMode).


Experiment Segment 6 - Understanding access check
* In Windows every object is a securable one. You can use the "!handle" and "!object"
* to dump unnamend and named objects
!object \BaseNamedObjects
* Just take a random object and analyze its Security Descriptor
!object <Object>
* Note that the Security descriptor is a FAST_REF object, hence its lower 4 bits should 
* be cleared out
dx ((_OBJECT_HEADER*)<ObjHdr>)->SecurityDescriptor & ~0xf
!sd <Security_Descriptor>
* PsGetSid can help you in understanding which SID corresponds to which user/group

* To dump the Access token we need to choose a process first
!process 0 0 <ProcessName>
!process <ProcObj> 1
!token <Token_Obj>
```
