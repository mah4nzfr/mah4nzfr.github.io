---
title: HTB RFlag - Challenge
date: 2025-06-23 20:42:00 +0330
categories: [Challenge]
tags: [hardware]
description: My writeup of RFlag from Hack The Box

---

After unzipping the file, we can only see a `.cf32` file so I decoded it and got a hex string.
```
$ rtl_433 -A signal.cf32
rtl_433 version 25.02 (2025-02-19) inputs file rtl_tcp RTL-SDR SoapySDR
[Input] Test mode active. Reading samples from file: signal.cf32
Detected OOK package    @0.220228s
Analyzing pulses...
Total count:  185,  width: 1837.12 ms           (459281 S)
Pulse width distribution:
 [ 0] count:  114,  width: 3608 us [3604;3624]  ( 902 S)
 [ 1] count:   71,  width: 7204 us [7200;7208]  (1801 S)
Gap width distribution:
 [ 0] count:   71,  width: 7172 us [7172;7180]  (1793 S)
 [ 1] count:  113,  width: 3576 us [3576;3584]  ( 894 S)
Pulse period distribution:
 [ 0] count:   57,  width: 10784 us [10780;10796]       (2696 S)
 [ 1] count:   42,  width: 14380 us [14376;14384]       (3595 S)
 [ 2] count:   85,  width: 7188 us [7184;7196]  (1797 S)
Pulse timing distribution:
 [ 0] count:  227,  width: 3592 us [3576;3624]  ( 898 S)
 [ 1] count:  142,  width: 7188 us [7172;7208]  (1797 S)
 [ 2] count:    1,  width: 72084 us [72084;72084]       (18021 S)
Level estimates [high, low]:  15985,    488
RSSI: -0.2 dB SNR: 30.3 dB Noise: -30.5 dB
Frequency offsets [F1, F2]:   -5928,      0     (-22.6 kHz, +0.0 kHz)
Guessing modulation: Manchester coding
view at https://triq.org/pdv/#AAB1030E081C14FFFF819191919191919191919191919191918080808090818080918090808180918091808080919191808091808080918090808081908191918091809180809081809190808080819180918080808090819180809081808090819081919081809081808091908190808180809081908180919080808081809081808091908081809081919080808081908180809081809081808080808090818080808090819081808080918080809180918080809180918080809190808080819255
Attempting demodulation... short_width: 3608, long_width: 0, reset_limit: 7184, sync_width: 0
Use a flex decoder with -X 'n=name,m=OOK_MC_ZEROBIT,s=3608,l=0,r=7184'
[pulse_slicer_manchester_zerobit] Analyzer Device
codes     : {256}2aaaaaaa0c4e4854427b52465f4834636b316e365f31735f6330306c2121217d
```

Then used python to convert it to ASCII:
```
>>> txt
'2aaaaaaa0c4e4854427b52465f4834636b316e365f31735f6330306c2121217d'
>>> bytes.fromhex(txt[8:])
b'\x0cNHTB{RF_H4ck1n6_1s_c00l!!!}'
```

Ever heard piece of cake? well this challenge was the embodiment of it.