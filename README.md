# Hex-editing the RIL blobs (RIL fix for LOS 17.1)

Note: this is a translation of the post on 4pda: 
https://4pda.to/forum/index.php?showtopic=209610&st=34560#entry93381502

Credits for translation goes to [LogosA](https://forum.xda-developers.com/member.php?u=5554397).

Briefly about the fix: changes to the Parcel class in libbinder broke the ABI for all blobs that link to libbinder and use the Parcel class.
The fix suggests to change blobs so that wherever a Parcel object is created, to allocate enough space on the stack for this object.

Original fix idea from [javelinanddart](https://forum.xda-developers.com/member.php?u=5795145)  ([i9300: Remove ashmem tracking hack](https://review.lineageos.org/c/LineageOS/android_device_samsung_i9300/+/167912)).

This fix was tested on the Galaxy S3, I decided to share it, because similar changes have been needed since Marshmallow, when the RIL was also broken on a bunch of devices.
If you have RIL blobs

<details>
  <summary>crashing in random places</summary>
  
  ```shell
01-29 14:17:11.681  2697  2697 I crash_dump32: performing dump of process 2048 (target tid = 2151)
01-29 14:17:11.691  2697  2697 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
01-29 14:17:11.691  2697  2697 F DEBUG   : LineageOS Version: '17.1-20200126-UNOFFICIAL-i9300'
01-29 14:17:11.691  2697  2697 F DEBUG   : Build fingerprint: 'samsung/m0xx/m0:4.3/JSS15J/I9300XXUGMJ9:user/release-keys'
01-29 14:17:11.691  2697  2697 F DEBUG   : Revision: '0'
01-29 14:17:11.691  2697  2697 F DEBUG   : ABI: 'arm'
01-29 14:17:11.693  2697  2697 F DEBUG   : Timestamp: 2020-01-29 14:17:11+0300
01-29 14:17:11.693  2697  2697 F DEBUG   : pid: 2048, tid: 2151, name: rild  >>> /vendor/bin/hw/rild <<<
01-29 14:17:11.693  2697  2697 F DEBUG   : uid: 1001
01-29 14:17:11.693  2697  2697 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
01-29 14:17:11.694  2697  2697 F DEBUG   : Cause: null pointer dereference
01-29 14:17:11.694  2697  2697 F DEBUG   :     r0  4c27213c  r1  00000000  r2  4c27215d  r3  00000000
01-29 14:17:11.694  2697  2697 F DEBUG   :     r4  0000000d  r5  00000000  r6  00000001  r7  00000000
01-29 14:17:11.694  2697  2697 F DEBUG   :     r8  4bc281a8  r9  4bc2a7f0  r10 4bbec93c  r11 0000000b
01-29 14:17:11.694  2697  2697 F DEBUG   :     ip  4ab300a0  sp  4c272188  lr  4ab0ac99  pc  4bb6fc2a
01-29 14:17:11.650  2053  2053 I tombstoned: type=1400 audit(0.0:168): avc: denied { write } for name="tombstones" dev="mmcblk0p12" ino=335874 scontext=u:r:tombstoned:s0 tcontext=u:object_r:system_data_file:s0 tclass=dir permissive=1
01-29 14:17:11.708  2697  2697 F DEBUG   :
01-29 14:17:11.708  2697  2697 F DEBUG   : backtrace:
01-29 14:17:11.708  2697  2697 F DEBUG   :       #00 pc 0002ec2a  /system/vendor/lib/libsec-ril.so (requestScreenState+70)
01-29 14:17:11.708  2697  2697 F DEBUG   :       #01 pc 0002184d  /system/vendor/lib/libsec-ril.so
01-29 14:17:11.708  2697  2697 F DEBUG   :       #02 pc 000a6c97  /apex/com.android.runtime/lib/bionic/libc.so (__pthread_start(void*)+20) (BuildId: eb3836dc9ac2643bb8a03f339fe9c540)
01-29 14:17:11.708  2697  2697 F DEBUG   :       #03 pc 000600e1  /apex/com.android.runtime/lib/bionic/libc.so (__start_thread+30) (BuildId: eb3836dc9ac2643bb8a03f339fe9c540)
  ```
  
  ```shell
  01-29 14:32:40.829  3287  3287 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
01-29 14:32:40.829  3287  3287 F DEBUG   : LineageOS Version: '17.1-20200126-UNOFFICIAL-i9300'
01-29 14:32:40.829  3287  3287 F DEBUG   : Build fingerprint: 'samsung/m0xx/m0:4.3/JSS15J/I9300XXUGMJ9:user/release-keys'
01-29 14:32:40.829  3287  3287 F DEBUG   : Revision: '0'
01-29 14:32:40.829  3287  3287 F DEBUG   : ABI: 'arm'
01-29 14:32:40.832  3287  3287 F DEBUG   : Timestamp: 2020-01-29 14:32:40+0300
01-29 14:32:40.832  3287  3287 F DEBUG   : pid: 3240, tid: 3282, name: rild  >>> /vendor/bin/hw/rild <<<
01-29 14:32:40.832  3287  3287 F DEBUG   : uid: 1001
01-29 14:32:40.832  3287  3287 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
01-29 14:32:40.832  3287  3287 F DEBUG   : Cause: null pointer dereference
01-29 14:32:40.832  3287  3287 F DEBUG   :     r0  05b826b5  r1  05b826b5  r2  00000000  r3  00000000
01-29 14:32:40.832  3287  3287 F DEBUG   :     r4  497b993c  r5  00000000  r6  00000031  r7  00000001
01-29 14:32:40.832  3287  3287 F DEBUG   :     r8  4a1e117c  r9  497f77f0  r10 497b993c  r11 00000000
01-29 14:32:40.832  3287  3287 F DEBUG   :     ip  00000000  sp  4a1e1028  lr  48ca104f  pc  4974646c
01-29 14:32:40.838  3287  3287 F DEBUG   : 
01-29 14:32:40.838  3287  3287 F DEBUG   : backtrace:
01-29 14:32:40.838  3287  3287 F DEBUG   :       #00 pc 0003846c  /system/vendor/lib/libsec-ril.so (checkRildReset+208)
01-29 14:32:40.838  3287  3287 F DEBUG   :       #01 pc 00024f0f  /system/vendor/lib/libsec-ril.so (requestRadioPower+114)
01-29 14:32:40.838  3287  3287 F DEBUG   :       #02 pc 0002184d  /system/vendor/lib/libsec-ril.so
01-29 14:32:40.838  3287  3287 F DEBUG   :       #03 pc 000a6c97  /apex/com.android.runtime/lib/bionic/libc.so (__pthread_start(void*)+20) (BuildId: eb3836dc9ac2643bb8a03f339fe9c540)
01-29 14:32:40.838  3287  3287 F DEBUG   :       #04 pc 000600e1  /apex/com.android.runtime/lib/bionic/libc.so (__start_thread+30) (BuildId: eb3836dc9ac2643bb8a03f339fe9c540)
```
  
</details>

it's possible that the reason is the ABI incompatibility, and this fix may help.

1. Load libsec-ril.so blob in any disassembler (on the example of IDA).
Wait until the functions analisys is finished, and look for the function android::Parcel::Parcel(void), aka _ZN7android6ParcelC1Ev.
We next will find the places where the function is called (by highlighting the beginning of the function and pressing "X"):
 
![](https://github.com/ChronoMonochrome/hacking_the_blobs/raw/master/1.png)

2. In the example there are only three places where this function is called from. Let's move on to the second place, RIL_onMultiClientRequestComplete (since this is a short function and easier to parse for example), we find the beginning of the function:

![](https://github.com/ChronoMonochrome/hacking_the_blobs/raw/master/2.png)
 
We are interested in the selected line,

```assembly
.text: 0001DC10 SUB SP, SP, # 0x34
```

This instruction allocates 0x34 bytes on the stack. Our goal is to change the instruction so that more bytes are allocated on the stack (how many exactly - on S3 it was enough to add 12 bytes, or, if you count 4 added bytes from the time of Oreo, then 16 bytes totally. The main thing is not to go too far and not allocate too much).

3. Remember the offset (0001DC10)
Copy the code (SUB SP, SP, # 0x34)
and paste it into any ARM assembler, for example
[http://shell-storm.org…format=inline#assembly](http://shell-storm.org/online/Online-Assembler-and-Disassembler/?inst=SUB+++++++++++++SP%2C+SP%2C+%230x34&arch=arm-t&as_format=inline#assembly)
(you need to choose the set of instructions accordingly, in this case we chose ARM Thumb)
In the IDA, go to the Hex view tab:
 
![](https://github.com/ChronoMonochrome/hacking_the_blobs/raw/master/3.png)

And we make sure that the same bytes (8D B0) are highlighted in the IDA that the online assembler gave us.
In online assembler, change the code to

```assembly
SUB SP, SP, # 0x40
```
(0x34 + 12 = 0x40)

Take the assembler output (under "Assembly - Little Endian"):
"\x90\xb0"

We see that only the first byte has changed. So far, we have found only one instruction at the beginning of the function, where bytes are allocated on the stack.
We also need to find an instruction at the end of the function that frees up those bytes on the stack.

4. Go to the end of the function
![](https://github.com/ChronoMonochrome/hacking_the_blobs/raw/master/4.png)

Again, we are interested in the highlighted line:
```assembly
.text: 0001DCF2 ADD SP, SP, # 0x34
```

As you can see, the instruction is similar to the one that was at the beginning of the function.
We need to change the code to the following

```assembly
.text: 0001DCF2 ADD SP, SP, # 0x40
```

(important! if in the first case 0x40 bytes were allocated, then you need to release exactly the same number of bytes)
We modify the code in exactly the same way as in step 3.

5. In step 2, we found three places where _ZN7android6ParcelC1Ev is called.
Similarly, you need to modify the remaining functions, as described in steps 3-4.

6. When we found all the offsets, we edit the blob with any hex editor in accordance with what the online assembler issued.
