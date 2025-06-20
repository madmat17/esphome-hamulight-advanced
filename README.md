# New Document

# ESP Home Hamulight Advanced

Implementation of the Hamulight protocol in ESPHome for emulating the Hamulight LED driver remote control.
This is a merge of my personal protocol analysis and [Auke de Jong's ESPHome implementation of the Hamulight protocol](https://github.com/aukedejong/esphome-hamulight).
It populates all Hamulight remote buttons as well as allows to customize the remote's ID used (in case multiple Hamulight LED drivers are to be controlled indipendently)


## The protocol


### Address / Header
First 2 bytes of the protocol is the remote's ID. The assumption (based on the the analysis of 2 different remotes) is that you can choose
ANY address here, since the driver/receiver can be trained for any Hamulight remote with that protocol.
<br/><br/>

### Remote Control's Commands
5 buttons of the remote are hard-coded, a capactivie touch field is used for seamless dimming

| Command         | Byte | Description                                                               |
| --------------- | ---- | ------------------------------------------------------------------------- |
| RFaddress       | var  | change the address as needed to match your physical remote                |
| RFpower         | `0x5F` | on/off toggle                                                             |
| RFbright100     | `0x59` | 100% brightness command + command for training the transformator/receiver |
| RFbright75      | `0x50` | 75% brightness command                                                    |
| RFbright50      |	`0x56` | 50% brightness command                                                    |
| RFbright25      |	`0x55` | 25% brightness command                                                    |
| RFslideRangeMin | `0x80` | lowest HEX value used by dimm slider                                      |
| RFslideRangeMax |	`0xFF` | highest HEX value used by dimm slider                                     |
| RFslideOffset   |	`0xA8` | needed for the offset in the slideValConv() function                      |
| RFslideSteps    | calc | steps from 0% to 100% dimm value (RFslideRangeMax - RFslideRangeMin + 1)  |
| RFslideStart    | calc | is the 0% dimm value position    (RFslideRangeMin + RFslideOffset)        |
<br/>

### Protocol logic / radio signal pattern
The implementation of the radio signal pattern is implemented dynamically to allow a definition of your own protocol variant.
Current approach is that each bit is always `HIGH` followed by a `LOW`in the duration of a specific length each.
First you need to know the smallest common divisor of all pulse lengths and define it micro seconds (e.g. 200uS).
Second you need to define the bits based on their `HIGH`/`LOW` durations (e.g. `{3, 1}` would be 3x 200uS = 600uS `HIGH` followed by 1x 200uS = 200uS `LOW`.
The sync sequence and start bit is to be defined in the array `startSequence`. The reading is the same as for the bit arrays, but a little longer to allow more complex code. **IMPORTANT:** The array itself begins with the first `HIGH` (can be changed by a leading `0` in the sequence). 

| Protocol element  | Value                     | Description                                                                                                                                                              |
| ----------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `basePulse`       | `200`                     | base pulse length in micro seconds                                                                                                                                       |
| `startSequence[]` | `{ 1,1,1,1,1,1,1,1,6,6 }` | pulse sequence for sync + RF start bit                                                                                                                                   |
| `Bit0[]`          | `{ 3,1 }`                 | pulse for bit 0 - currently only an array of 2 values is supported!                                                                                                      |
| `Bit1[]`          | `{ 1,3 }`                 | pulse for bit 1 - currently only an array of 2 values is supported!                                                                                                      |
| `codeSequence[64]`| variable                  | Array of 64 values (64 high/low changes for 32 bit signal); any changes to the `Bit0[]`/`Bit1[]` array size need to be reflected in the code sequence array size as well |
| `signalRepetition`| `6`                       | how many times the signal should be transmitted (usally 4-6 times for transmission robustness)                                                                           |
<br/>

### Checksum
Calculated automatically.
The checksum in the Hamulight protocol is a simple byte addition with an offset of minus `0x53`.
<br/>

## Pairing process
Disconnect and reconnect LED driver from mains and send the "max brightness" command within the first 10 seconds.
