# TEST_IXXAT_ADWIN_APPOLON

Communication tests between C# and ADBasic.

With these two pieces of code, it's possible to send all necessary event codes and values from c# to ADBasic using CAN bus :

* BEGIN SESSION
* BEGIN TRIAL
* START with ACCEL value (IEEE754 32 bits)
* START with SPEED value (IEEE754 32 bits)
* START with DURATION value (IEEE754 32 bits)
* STOP
* RESET
* END TRIAL
* END SESSION

CSharp.md is C# code

ADBasic.md is ADBasic code
