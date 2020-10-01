## AirPortAtheros40 patches

1. Fix Wi-Fi region domain

__Problem__. Non-Apple Atheros cards may not have a valid Wi-Fi region domain in firmware. As a result, in system log there are  multiple lines like this:
```
ATHR: unknown locale 21
```
Or another locale like 60, 61, 809c and so on. 

Also macOS System Report application shows  Atheros Wi-Fi card locale as `Unknown` or as empty field:
```
Card Type:    AirPort Extreme  (0x16C8, ...)
...
Locale: Unknown (or empty field)
...
```

The reason is that system kext (AirPortAtheros40) has limited list of valid Wi-Fi region domains. By reverse engeenering the following list of valid Wi-Fi region domains can be obtained:
|Region hex code|macOS locale string|Comment|
|-|-|-|
|37|ETSI|ETSI1_WORLD|
|5E||APL9_WORLD|
|64|FCC|WOR4_WORLD|
|65||WOR5_ETSIC|
|6A||WORA_WORLD|
|8D||MKK7_MKKA2|

Some codes available [here](https://github.com/freebsd/freebsd/blob/1d6e4247415d264485ee94b59fdbc12e0c566fd0/sys/dev/ath/ath_hal/ah_regdomain/ah_rd_regenum.h)

__Solution__. Patch kext to pass custom region domain value (one from the list above).

Function `_wlan_get_regdomain` return region domain from card firmware (from some structure).

Re-interpreted code in c:
```c
int _wlan_get_regdomain(int arg0) {
    rax = *(int16_t *)(arg0 + 0x4dc) & 0xffff; // Value from some internal structure
    return rax;
}
```

And dissasembled one:
```
_wlan_get_regdomain:
push       rbp
mov        rbp, rsp
movzx      eax, word [rdi+0x4dc] # Value from some internal structure
pop        rbp
ret
```

So, rewrite function to return valid constant value of region domain.
```
_wlan_get_regdomain:
push       rbp
mov        rbp, rsp
mov        eax, 0x64 # Value from valid region domain list (FCC)
pop        rbp
ret
```

Patch and replace following hex code in kext.
```
Find: 0F B7 87 DC 04 00 00
Replace: B8 XX 00 00 00 90 90
```
Replace XX with 64, 37, 5E, 65, ... (from valid region domain list above).

It's not a universal patch, but AirPortAtheros40 kext is a legacy solution. You can use one version for modern macOS .
