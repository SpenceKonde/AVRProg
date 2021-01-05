# AVRProg
Objective: AVRProg.h library to include in Arduino sketch which will be able to program another AVR via ISP or UPDI

## Target Hardware
* AVR Dx-series
* megaAVR 0-series (will get tinyAVR 0/1-series "for free" here too) 
* ATmega328, ATmega1284p, ATmega2560, ATmega32u4
* tinyAVR not a priority.
  * This will be much easier if we have a dedicated buffer than can fit 1 page of flash 


## Questions

### ISP and UPDI modes
Each mode will need separate implementation of the functions. Ideally we want as much as possible to remain the same, though, and we want a single sketch to be able to use more than one programming mode if they so choose.... 

I mean, once this much is in... there's a clear structure by which others could a new "mode" to support, say, STK500 programming or something, right?

### Data format, function signatures
My first thought is that we would have some staticly defined buffers of the maximum sizes we were willing to work with... 
```
// Both of these get memset() to 0xFF on startup...
uint8_t ReadBuffer[512];  // Largest page size is curently the AVR Dx-series
uint8_t WriteBuffer[512]; // Also used for EEPROM - but see below, I think for EEPROM, we ought to have a union with a 16-bit datatype.
uint8_t SignatureRow[64]; // Most of this is empty (in theory - it looks like there may be undocumented stuff here that could act as a
                          // chip ID on classic parts... or maybe just some of them...
                          // But within classic parts, there are a few wacky parts that have something at, 
                          // say, byte 0x2C (t841) in the "device signature impression table"
                          // and the tinyAVR parts do store their cal constants above 32k. 
uint8_t Fuses[16];        // Dx-series defines size of fuses at 16-bytes, read fuses stored here.


```


#### Read functions
* For what cases (if any) should we permit unaligned reads? 
 * All cases - why shouldn't people read whatever bytes they want?
 * Never for flash. Consistency is king; for part with page size 2^N, the lower N bits are always ignored in address fields).
 * Never when reading multiple bytes from flash. `readFlashByte()` function to allow checking specific bytes.
* Should EEPROM reads be required to be aligned? 
 * There's something to be said for dumping the largest chunk you can to the buffer... but on paged-EEPROM devices, also something to be said for just dumping one page, so the application using it can examine a page at a time, decide what it needs to change, and do that gracefully. I suppose there could be two cals, one to read a (potentially as large as the buffer) chunk of EEPROM, and the other to just read a page... either way, it would return number of bytes read, or negative error status...
* I don't see any reason not to try to read the whole signature row when we read the signature. Sure, most of it doesn't matter, but why not grab  it all - unless the overhead is severe, IMO grab the full sigrow/device signature impression table. 
* Same goes for reading fuses. 
 
#### Write functions
* Flash write functions need a second argument for options, eg (on UPDI parts) erase page first (if modern AVR) or at least return error instead of shitting on whatever was already there if it's not empty, and to re-read what was written and validate it against what was in the write buffer.
* EEPROM harder to do consistently across devices, since they're so different...
 * Classic AVRs have EEPROM page of 4 or 8 bytes.
 * tinyAVR 0/1/2-series and megaAVR 0-series have page sizes that are a substantial fraction of the size of the EEPROM.
 * Dx-series EEPROM doesn't have any of that - you just write the the ERASE WRITE command active, and it writes the byte you write to.
 * This sort of buffer could be mimiced though! 
* For EEPROM, only bytes "written" are erased and written to the EEPROM. If we want to avoid unnecessary EEPROM erase/write cycles, we need to track which addresses in the buffer get written to... how about a union with an array with half as many elements of an 16-bit datatype... when we clear the write buffer after a successful write, both bytes would be set (ie, set to 0xFFFF), while the application would (hopefully) be stuffing uint8_t's in there. So we check the high byte of each element, if if's 0, then they overwrite that element with an 8-bit value, if not, then they didn't and we don't write that one...).

  
## Proposed API
Note above question which influences how these might be named. I am inclined to outlaw unaligned read fros
`init(...)` - initialize an ISP programming interface; will require passing arguments that
`setPart(uint8_t sig0, uint8_t sig1 uint8_t sig2)` - set the device (using the device ID); sig0 is unnecessary as it's always 0x1E. 
`readSignature()` - Read the signature
  * Should this get the device ID signature only? Or should it read the whole thing (classic AVRs this includes cal bytes, modern AVRs this includes 
`int8_t readFuses()` - Read the fuses, stores in Fuses array. Sets the rest of the array to 0x00, and returns either the number of fuses read. 
`uint8_t readLock()` - Classic AVR: Return the value of the lockbits (a byte wherein the two high bits are unused; on some parts, more bits are unused). Modern AVR: Return of 1 is unlocked, 0 is locked. There's no middle ground on these parts. 
`int16_t readEEPROM(uint16_t address, [uint16_t length?])` - Read the specified amount of EEPROM memory of target? Maybe it's better to just read EEPROM page ( 
`int16_t readFlash(uint32_t address)` - Read the the entire page containing said address, place it in ReadBuffer. Returns number of bytes read (should be the page size - but returning that as success would probably make generic flash dumpers easier to write...) or an error (which would be a negative number. 
`int16_t readEEPROMByte(uint16_t address)` - read a single byte at address. Returns either positive value between 0 and 256 on success, or negative error code if failure. 
`int16_t writeFlash(uint32_t address, uint8_t options)` - write the buffer. Return number of bytes written. Maybe an option should determine whether this counts the full size of buffer, or just non-0xFF bytes (maybe it should even be the same option that controls whether it cares whether there's already data there? That would make sense...)
`int16_t writeEEPROMPage(uint16_t address)` - write page of EEPROM. How to do this for Dx-series? Maybe it would be the same as writeEEPROMBuffer()? Returns number of bytes written, which will be equal to or less than the number of bytes in a page...
`int16_t writeEEPROMBuffer(uint16_t address)` - write up to the size of the buffer (256b if it's the 512b buffer and the 2-byte datatype trick described above), ignoring the low bits of the address given.
`int8_t writeEEPROMByte(uint16_t address, uint8_t value)` - immediately write a byte to the EEPROM. Returns 1 on success, or negative error code. 
`int8_t writeFuse(uint8_t fuse, uint8_t value)` - write *fuse* to *value*.
`int8_t writeLock(uint8_t value)` write the lockbits. On modern AVRs, value is ignored - any call to this will lock the chip.

You get the idea. Naturally the modern AVRs would also get a USERROW read and write function; note that tinyAVR and megaAVR can rewrite bytewise in userrow, Dx-series can only  erase the whole tyhing... so it ought to have an option to control what it should do if it finds that there's something already written to a byte it is told to write... (return error and don't write, write over it knowing that the data won't match what was written, or read in the parts of the USERROW trhat aren't being written, erase it, and write the rewrite it.... 

## Shortcut to determine flash, pagesize from signatures
```
Flash = 1024 << (SIGNATURE_1 & 0x0F); 
```

There are only three exceptions to the rule of flash size and signature: tiny4 and tiny5 (0x90, 0x09 and 0x0A) have 512b of flash, not the expected 1k, the ATmega40 - a 40K flash chip with a 32k signature (what a weirdo flash size! Then again, that whole part is bizarre - also utterly uninteresting for most of us, unless you want to make a battery management system, and are most comfortable programming AVRs (in that case, it's just what the doctor ordered)... And the 4808/4809: 0x96, 0x50/0x51. The sequential SIGNATURE_2 byte would have been 0x10 or 0x11. If they ever release a non-xmega AVR with a 3/4ths flash size like that again, I guess we'll find out if thats a well-reasoned pattern or the result of a slapdash effort to figure out how to signify that in the signature. In support of the "slapdash" theory, the xmega numbering started from 0x40... and, ah, there's already a `0x1E9651` - the ATxmega64b3...

The low six bits of the final signature byte seem to be simply the order in which the parts were designed... including parts that got canceled very close to release. It does look like with the release of the "modern" AVR parts in 2016, they cheated in some sizes, and skipped a few numbers. The high bit in SIGNATURE_2 indicates that there's something weird about the part.

Anyway; simple rules can be used to determine the flash size, page size, and so on from the device ID - basically a list of parts with exceptions, plus the general rules...

## Parting thought...
It would sure make it easy to make standalone programmers... One of the incredibly frustrating things about jtag2updi is that it is so far from an Arduino sketch that adapting it to a standlone programmer is nigh impossible. Which kinda sucks, and frankly is sort of holding back the UPDI ecosystem... This is the sort of thing I would love to do if I didn't have three cores to work on already...
