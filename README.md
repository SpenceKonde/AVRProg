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
uint8_t WriteBuffer[512]; // Also used for EEPROM.
uint8_t SignatureRow[64]; // Most of this is empty... 
                          // but even on classic parts, there are a few wacky parts that have something at, 
                          // say, byte 0x2C (t841) in the "device signature impression table"
uint8_t Fuses[16];        // Dx-series defines size of fuses at 16-bytes, though most are unused on existing parts.



```


#### Read functions
For what cases (if any) should we permit unaligned reads? 
* All cases - why shouldn't people read whatever bytes they want?
  * Never for flash. Consistency is king; for part with page size 2^N, the lower N bits are always ignored in address fields).
  * Never when reading multiple bytes from flash. `readFlashByte()` function to allow checking specific bytes.
* EEPROM is messy 
  * Classic AVRs have EEPROM page of 4 or 8 bytes. 
  
  tinyAVR 0/1/2-series and megaAVR 0-series have page sizes that are a substantial fraction of the size of the EEPROM... And the Dx-series doesn't have a concept of pages. 
  
## Proposed API
Note above question which influences how these might be named. I am inclined to outlaw unaligned read fros
`init(...)` - initialize an ISP programming interface; will require passing arguments that
`setPart(uint8_t sig0, uint8_t sig1 uint8_t sig2)` - set the device (using the device ID); sig0 is unnecessary as it's always 0x1E. 
`readSignature()` - Read the signature
  * Should this get the device ID signature only? Or should it read the whole thing (classic AVRs this includes cal bytes, modern AVRs this includes 
`readFuses()` - Read the fuses. For modern parts, read the 10 bytes that could conceivably be fusebytes. 
`readLock()` - Classic AVR: Return the value of the lockbits (a byte wherein the two high bits are unused; on some parts, more bits are unused). Modern AVR: Return of 1 is unlocked, 0 is locked. There's no middle ground on these parts. 
`readEEPROM(uint16_t address, [uint16_t length?])` - Read the specified amount of EEPROM memory of target? Maybe it's better to just read EEPROM page ( 
`readFlash(uint16_t address)` - Read the the entire page containing said address, place it in 
`
` 

## Shortcut to determine flash, pagesize from signatures
```
Flash = 1024 << (SIGNATURE_1 & 0x0F); 
```

There are only three exceptions to the rule of flash size and signature: tiny4 and tiny5 (0x90, 0x09 and 0x0A) have 512b of flash, not the expected 1k, the ATmega40 - a 40K flash chip with a 32k signature (what a weirdo flash size! Then again, that whole part is bizarre - also utterly uninteresting for most of us, unless you want to make a battery management system, and are most comfortable programming AVRs (in that case, it's just what the doctor ordered)... And the 4808/4809: 0x96, 0x50/0x51. The sequential SIGNATURE_2 byte would have been 0x10 or 0x11. If they ever release a non-xmega AVR with a 3/4ths flash size like that again, I guess we'll find out if thats a well-reasoned pattern or the result of a slapdash effort to figure out how to signify that in the signature. In support of the "slapdash" theory, the xmega numbering started from 0x40... and, ah, trhere's already a `0x1E9651` - the ATxmega64b3


The low six bits of the final signature byte seem to be simply the order in which the parts were designed... including parts that got canceled very close to release. It does look like with the release of the "modern" AVR parts in 2016, they cheated in some sizes, and skipped a few numbers. The 
