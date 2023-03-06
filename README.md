## mp4v2_mp4track_poc

### Build project

[Address](https://github.com/enzo1982/mp4v2)

#### CMake

Run cmake . && make in the MP4v2 project base folder

### POC

use poc repleace target file.


### Problem
There has a FPE(Floating Point Exception) in mp4track.cpp:999:46 in mp4v2::impl::MP4Track::GetSampleFileOffset(unsigned int), Attackers cause denial of service through carefully constructed malicious files.

```
        sampleId - ((sampleId - firstSample) % samplesPerChunk);
```
Because malicious file causes samplesPerChunk == 0, It is FPE.

### ASAN report

```bash
(base) ➜  build git:(main) ✗ ./mp4extract out/default/crashes/id:000000,sig:08,src:001076,time:147809374,execs:155756872,op:havoc,rep:8
./mp4extract version 2.1.2
ReadAtom: "out/default/crashes/id:000000,sig:08,src:001076,time:147809374,execs:155756872,op:havoc,rep:8": invalid atom size, extends outside parent atom - skipping to end of "" "moov" 12337 vs 12050
ReadAtom: "out/default/crashes/id:000000,sig:08,src:001076,time:147809374,execs:155756872,op:havoc,rep:8": invalid atom size, extends outside parent atom - skipping to end of "stbl" "J" 1212684099 vs 5988
UndefinedBehaviorSanitizer:DEADLYSIGNAL
==2270667==ERROR: UndefinedBehaviorSanitizer: FPE on unknown address 0x7f4c8a4317b9 (pc 0x7f4c8a4317b9 bp 0x000000000000 sp 0x7ffc56e8d660 T2270667)
    #0 0x7f4c8a4317b9 in mp4v2::impl::MP4Track::GetSampleFileOffset(unsigned int) /root/mp4v2/src/mp4track.cpp:999:46
    #1 0x7f4c8a42fc1a in mp4v2::impl::MP4Track::ReadSample(unsigned int, unsigned char**, unsigned int*, unsigned long*, unsigned long*, unsigned long*, bool*, bool*, unsigned int*) /root/mp4v2/src/mp4track.cpp:306:27
    #2 0x7f4c8a417c53 in mp4v2::impl::MP4File::ReadSample(unsigned int, unsigned int, unsigned char**, unsigned int*, unsigned long*, unsigned long*, unsigned long*, bool*, bool*, unsigned int*) /root/mp4v2/src/mp4file.cpp:3119:41
    #3 0x7f4c8a3f5aca in MP4ReadSample /root/mp4v2/src/mp4.cpp:3050:36
    #4 0x42887b in ExtractTrack(void*, unsigned int, bool, unsigned int, char*) /root/mp4v2/util/mp4extract.cpp:223:14
    #5 0x428376 in main /root/mp4v2/util/mp4extract.cpp:175:13
    #6 0x7f4c89dcd082 in __libc_start_main /build/glibc-SzIz7B/glibc-2.31/csu/../csu/libc-start.c:308:16
    #7 0x40679d in _start (/root/mp4v2/build/mp4extract+0x40679d)

UndefinedBehaviorSanitizer can not provide additional info.
SUMMARY: UndefinedBehaviorSanitizer: FPE /root/mp4v2/src/mp4track.cpp:999:46 in mp4v2::impl::MP4Track::GetSampleFileOffset(unsigned int)
==2270667==ABORTIN
```


I use gdb debug this bug:

### GDB report

```bash
gdb-peda$ r
Starting program: /home/ubuntu/Desktop/test_mp4v2/mp4v2/mp4extract id_000000,sig_08,src_001076,time_147809374,execs_155756872,op_havoc,rep_8
/home/ubuntu/Desktop/test_mp4v2/mp4v2/mp4extract version 2.1.2
ReadAtom: "id_000000,sig_08,src_001076,time_147809374,execs_155756872,op_havoc,rep_8": invalid atom size, extends outside parent atom - skipping to end of "" "moov" 12337 vs 12050
ReadAtom: "id_000000,sig_08,src_001076,time_147809374,execs_155756872,op_havoc,rep_8": invalid atom size, extends outside parent atom - skipping to end of "stbl" "J" 1212684099 vs 5988

Program received signal SIGFPE, Arithmetic exception.
[----------------------------------registers-----------------------------------]
RAX: 0x0 
RBX: 0x0 
RCX: 0x0 
RDX: 0x0 
RSI: 0x0 
RDI: 0x555555590b10 --> 0x555555590ad0 --> 0x0 
RBP: 0x7fffffffd9f0 --> 0x7fffffffdad0 --> 0x7fffffffdb30 --> 0x7fffffffdbc0 --> 0x7fffffffde80 --> 0x7fffffffdef0 (--> ...)
RSP: 0x7fffffffd9b0 --> 0x100000000 
RIP: 0x7ffff7f532a5 (<_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+131>:	div    DWORD PTR [rbp-0x14])
R8 : 0x0 
R9 : 0x3 
R10: 0x7ffff7e6f085 ("_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj")
R11: 0x7ffff7f53222 (<_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj>:	endbr64)
R12: 0x555555556530 (<_start>:	endbr64)
R13: 0x7fffffffdfe0 --> 0x2 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x7ffff7f5329a <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+120>:	mov    eax,DWORD PTR [rbp-0x3c]
   0x7ffff7f5329d <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+123>:	sub    eax,DWORD PTR [rbp-0x18]
   0x7ffff7f532a0 <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+126>:	mov    edx,0x0
=> 0x7ffff7f532a5 <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+131>:	div    DWORD PTR [rbp-0x14]
   0x7ffff7f532a8 <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+134>:	mov    edx,eax
   0x7ffff7f532aa <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+136>:	mov    eax,DWORD PTR [rbp-0x1c]
   0x7ffff7f532ad <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+139>:	add    eax,edx
   0x7ffff7f532af <_ZN5mp4v24impl8MP4Track19GetSampleFileOffsetEj+141>:	mov    DWORD PTR [rbp-0x10],eax
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffd9b0 --> 0x100000000 
0008| 0x7fffffffd9b8 --> 0x555555592dd0 --> 0x7ffff7fb2d00 --> 0x7ffff7f500c4 (<_ZN5mp4v24impl8MP4TrackD2Ev>:	endbr64)
0016| 0x7fffffffd9c0 --> 0x2 
0024| 0x7fffffffd9c8 --> 0xefe285c4a1432b00 
0032| 0x7fffffffd9d0 --> 0x100000000 
0040| 0x7fffffffd9d8 --> 0x1 
0048| 0x7fffffffd9e0 --> 0x0 
0056| 0x7fffffffd9e8 --> 0x7ffff7fc7000 --> 0x7ffff7df3000 --> 0x10102464c457f 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGFPE
0x00007ffff7f532a5 in mp4v2::impl::MP4Track::GetSampleFileOffset(unsigned int) () from /home/ubuntu/Desktop/test_mp4v2/mp4v2/libmp4v2.so.2

```



![image-20230306185323566](C:\Users\root\AppData\Roaming\Typora\typora-user-images\image-20230306185323566.png)
