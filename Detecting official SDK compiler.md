# Detecting official SDK compiler

According to page 118 of *SHC Manual (Hitachi C Compiler for SuperH)*,
the compiler implicitly defines macros `__HITACHI__` and
`__HITACHI_VERSION__` in the format *aabb*, where *aa* is version and *bb* is
revision.  
Examples provided were:
```
#define __HITACHI_VERSION__ 0x0501 /* Version 5.1C */
#define __HITACHI_VERSION__ 0x0600 /* Version 6.0 */
```

Also, the page references several compiler toggles that can be "relayed" to
the source code via macros, such as *cpu type, endianness, double=float*, etc.

From this information, one can write code that can be compiled either with
GCC or SHC (Casio Official SDK).  
Example:
```
if defined(__HITACHI__) && !defined(__GNUC__)
/* code for official sdk here */
#elif defined(__GNUC__)
/* code for gcc here */
#endif
```
