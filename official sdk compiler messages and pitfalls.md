# SHC/official SDK Error/Warning messages and pitfalls

Here's an attempt to list the frequent error/warning messages when
using the official SDK.

## Compiler (Error/Warning) Messages
Most of the error messages encountered can be deciphered with the *SHC Manual*,
page 781 (Section 12 Compiler Error Messages).

## Warning C1016 (W) Argument mismatch

The following project sample provided by the Casio SDK compiles without issue:

```
 1: /*****************************************************************/
 2: /*                                                               */
 3: /*   CASIO fx-9860G SDK Library                                  */
 4: /*                                                               */
 5: /*   File name : [ProjectName].c                                 */
 6: /*                                                               */
 7: /*   Copyright (c) 2006 CASIO COMPUTER CO., LTD.                 */
 8: /*                                                               */
 9: /*****************************************************************/
10: #include "fxlib.h"
11:
12:
13: //****************************************************************************
14: //  AddIn_main (Sample program main function)
15: //
16: //  param   :   isAppli   : 1 = This application is launched by MAIN MENU.
17: //                        : 0 = This application is launched by a strip in eACT application.
18: //
19: //              OptionNum : Strip number (0~3)
20: //                         (This parameter is only used when isAppli parameter is 0.)
21: //
22: //  retval  :   1 = No error / 0 = Error
23: //
24: //****************************************************************************
25: int AddIn_main(int isAppli, unsigned short OptionNum)
26: {
27:   unsigned int key;
28:
29:   Bdisp_AllClr_DDVRAM();
30:
31:   locate(1,4);
32:   Print((unsigned char*)"This application is");
33:   locate(1,5);
34:   Print((unsigned char*)" sample Add-In.");
35:
36:   while(1){
37:   GetKey(&key);
38:   }
39:
40:   return 1;
41: }
42:
43:
44:
45:
46: //****************************************************************************
47: //**************                                              ****************
48: //**************                 Notice!                      ****************
49: //**************                                              ****************
50: //**************  Please do not change the following source.  ****************
51: //**************                                              ****************
52: //****************************************************************************
53:
54:
55: #pragma section _BR_Size
56: unsigned long BR_Size;
57: #pragma section
58:
59:
60: #pragma section _TOP
61:
62: //****************************************************************************
63: //  InitializeSystem
64: //
65: //  param   :   isAppli   : 1 = Application / 0 = eActivity
66: //              OptionNum : Option Number (only eActivity)
67: //
68: //  retval  :   1 = No error / 0 = Error
69: //
70: //****************************************************************************
71: int InitializeSystem(int isAppli, unsigned short OptionNum)
72: {
73:   return INIT_ADDIN_APPLICATION(isAppli, OptionNum);
74: }
75:
76: #pragma section
```

Although if you change line 32 from:  
```
Print((unsigned char*)"This application is");
```
to:
```  
Print("This application is");
```

The compiler will warn about:
```
Executing Hitachi SH C/C++ Compiler/Assembler phase

set SHC_INC=C:\Program Files\CASIO\fx-9860G SDK\OS\SH\include
set PATH=C:\Program Files\CASIO\fx-9860G SDK\OS\SH\bin
set SHC_LIB=C:\Program Files\CASIO\fx-9860G SDK\OS\SH\bin
set SHC_TMP=Z:\home\hilbert\projects\CASIO\testrun\DEMO1\Debug
"C:\Program Files\CASIO\fx-9860G SDK\OS\SH\bin\shc.exe" -subcommand=C:\users\hilbert\Temp\hmkcc34.tmp
Z:\home\hilbert\projects\CASIO\testrun\DEMO1\DEMO1.c(32) : C1016 (W) Argument mismatch

Executing Hitachi OptLinker04 phase

"C:\Program Files\CASIO\fx-9860G SDK\OS\SH\bin\Optlnk.exe" -subcommand=C:\users\hilbert\Temp\hmkccf0.tmp

Optimizing Linkage Editor Completed

HMAKE MAKE UTILITY Ver. 1.1
Copyright (C) Hitachi Micro Systems Europe Ltd. 1998
Copyright (C) Hitachi Ltd. 1998


	Make proccess completed

"Z:\home\hilbert\projects\CASIO\testrun\DEMO1\DEMO1.G1A" was created.

Build has completed.
```

This is due to the declaration of the `Print()` function being:
```
void Print(const unsigned char *str);
```

According to *SHC Manual*, page 783:
> C1016 (W) Argument mismatch  
The data type assigned to a pointer specified as a formal parameter in a prototype declaration
differs from the data type assigned to a pointer used as the corresponding actual parameter in a
function call. Uses the internal representation of the pointer used for the function call actual
parameter.

It essentially means that the pointer types are different.  
To fix this: cast strings as `(unsigned char *)` for `Print()` function in this case.
(match the argument type).
