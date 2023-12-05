# Ramdisk Checksum Verification

During [franken DSM attempts](../VDSM/franken-dsm.md) we discovered a DSM's kernel security feature. It doesn't seem to
be aimed to stop anybody from running DSM on unofficial hardware/HV, but rather to ensure the larger effort of making
sure the ***existing installation*** isn't damaged/tampered with. Giving the circumstances (DS hardware is a more 
embedded system than a PC) we will say it's probably to make sure a flash failure will not cause random code to be 
executed.

Kind of by accident checksum stops easy debugging of DSM on unofficial hardware.

### What this does to the system?
When the ramdisk image (`rd.gz`) loaded into the kernel is modified, its checksum/signature changes. This causes kernel 
to **deliberately**:
  - disable `mount(MS_MOVE)` syscall
  - disable `mount(MS_BIND)` syscall
  - enable enforcement of kernel modules signatures


### Implementation
The check is done just at the beginning of the initial rootfs (=ramdisk) unpacking:

  ```c
  # init/initramfs.c
  static int __init populate_rootfs(void)
  {
      char *err = unpack_to_rootfs(__initramfs_start, __initramfs_size);
      if (err)
          panic(err);     
  
  #ifdef MY_ABC_HERE
      //....
  
      if (hydro_sign_verify(sig, initrd_start, rd_len, ctx, pk)) {
          ramdisk_check_failed = true;
          printk(KERN_ERR "ramdisk corrupt");
      } else {
          ramdisk_check_failed = false;
          initrd_end -= hydro_sign_BYTES;
      }
  #endif 
  ```

In essence the whole ramdisk is verified, so it cannot be modified (or so they say ;)). This will cause 
*"ramdisk corrupt"* error to be present in `dmesg` output, while the boot continues "normally":

  ```
  DiskStation> dmesg | grep ramdisk  
  [    0.577409] ramdisk corrupt
  ```


#### `MS_MOVE` and `MS_BIND` checks
This flag is then checked in mount syscall implementation:
  ```c
  # fs/namespace.c:do_mount()
  //...
    else if ((flags & MS_BIND) && ramdisk_check_failed)
          retval = -EPERM;
  //...
    else if ((flags & MS_MOVE) && ramdisk_check_failed)
          retval = -EPERM;
  ```


#### Modules verification
Then the flag enabled a pesky modules signature verification:
  ```c
  # kernel/module.c
  #ifdef CONFIG_MODULE_SIG
  static int module_sig_check(struct load_info *info, int flags)
  {
      int err = -ENOKEY;
      const unsigned long markerlen = sizeof(MODULE_SIG_STRING) - 1;
      const void *mod = info->hdr;
  
  #ifdef MY_ABC_HERE
      sig_enforce |= ramdisk_check_failed;
  #endif  
  ```

Normally the kernel is built with signatures verification enabled (`CONFIG_MODULE_SIG=y`) but it is not enforced by 
default. In such state all modules signed by any of the keys present in the kernel (i.e. Synology's keys) can be loaded 
without problem. Theoretically you *CAN* load any other modules if you can get them into the kernel. While we didn't try 
you could possibly download a module via `curl` and do an `insmod foo.ko`. However, as soon as the ramdisk (`rd.gz`) is 
modified in any way (e.g. adding new modules) the signature check will not allow modules to load:

  ```
  DiskStation> insmod hello.ko
  insmod: can't insert 'hello.ko': Required key not available
  ```


### How does it play with loaders?
Requires a ([tricky binary patching](README.md#kernel-binary-patching)).


### Recreating ramdisks
Unpacking and repacking ramdisks is usually pretty straight forward:

```shell
# unpack ramdisk \/
mkdir "ramdisk" ; cd "ramdisk"
xz -dc < ../rd.gz | cpio -idmv

# make any modifications and recompress \/ 
find . 2>/dev/null | cpio -o -H newc -R root:root | xz -9 --format=lzma > ../rd.gz
```

However, works with only SOME kernels modified by Syno. Don't ask us why but they broke compressed ramdisks to the point
that using standard tools we weren't able to make them work again. For excample `apollolake` Linux v4.4.x from 25556 
release contains the following code in `init/initramfs.c`:
```c
#ifdef MY_ABC_HERE
    if (my_inptr == 0) {
        printk(KERN_INFO "decompress cpio completed and skip redundant lzma\n");
        break;
    }
#endif /* MY_ABC_HERE */
```

Attempting to boot with `lzma`-compressed ramdisk causes `unexpected EOF` error while using `gzip` produces read errors.
However, using uncompressed image works perfectly (but due to size it has to be placed on a second partition):

```shell
# make any modifications and recompress \/ 
find . 2>/dev/null | cpio -o -H newc -R root:root > ../rd.cpio

# use "initrd hd(0,1)/rd.cpio" instead of "initrd /rd.gz" in your grub config
```

#### Kernel patches
  - **DS3615xs** with **25556** kernel  
    *You can use the [ramdisk check autopatcher](../tools/patch-ramdisk-check.php) to apply the same changes 
    automatically.*

    ```
    c6 05 52 2e 0b 00 01       MOV        byte ptr [.....],0x1
  
    to
  
    c6 05 52 2e 0b 00 00       MOV        byte ptr [.....],0x0
    ```
  
  - **DS3617xs** with **25556** kernel  
    *You can use the [ramdisk check autopatcher](../tools/patch-ramdisk-check.php) to apply the same changes
    automatically.*

    ```
    //tbd (do a diff dumbass since you've lost the patch)
    ```

  - **DS918+** with **25556** kernel  
    *You can use the [ramdisk check autopatcher](../tools/patch-ramdisk-check.php) to apply the same changes
    automatically.*

    ```
    74 17                   JZ   LAB_ffffffff81893775    ; ramdisk check OK
    48 c7 c7 a2 71 71 81    MOV  RDI,0xffffffff817171a2  ; ramdisk corrupt str.
               =to=
    eb 17                   JMP  LAB_ffffffff81893775    ; ramdisk check always OK
    48 c7 c7 a2 71 71 81    MOV  RDI,0xffffffff817171a2  ; ramdisk corrupt str.
    ```

