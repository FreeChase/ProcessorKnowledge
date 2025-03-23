
## 中断触发详细行为

简而言之，动作如下
 1. 压栈保存通用寄存器
    ```lua
    MemA[frameptr,4] = R[0];
    MemA[frameptr+0x4,4] = R[1];
    MemA[frameptr+0x8,4] = R[2];
    MemA[frameptr+0xC,4] = R[3];
    MemA[frameptr+0x10,4] = R[12];
    MemA[frameptr+0x14,4] = LR;
    MemA[frameptr+0x18,4] = ReturnAddress(ExceptionType);
    MemA[frameptr+0x1C,4] = (XPSR<31:10>:frameptralign:XPSR<8:0>);

    
    习惯用法，R0-R12通用读写寄存器，R13-SP,R14-LR,R15-PC
    ```
 1. 执行中断函数

**详细伪代码如下：**

**The ExceptionEntry() pseudocode function describes the exception entry behavior:**
```lua
// Exception Number Enumeration
// ============================
constant integer Reset = 1;
constant integer NMI = 2;
constant integer HardFault = 3;
constant integer MemManage = 4;
constant integer BusFault = 5;
constant integer UsageFault = 6;
constant integer SVCall = 11;
constant integer DebugMonitor = 12;
constant integer PendSV = 14;
constant integer SysTick = 15;
// ExceptionEntry()
// ================
ExceptionEntry(integer ExceptionType)
// NOTE: PushStack() can abandon memory accesses if a fault occurs during the stacking
// sequence.
// Exception entry is modified according to the behavior of a derived exception,
// see DerivedLateArrival() and associated text.
    PushStack(ExceptionType);
    ExceptionTaken(ExceptionType);
```

**The definitions of the PushStack() and ExceptionTaken() pseudocode functions are:**
```lua
// PushStack()
// ===========
PushStack(integer ExceptionType)
    if HaveFPExt() && CONTROL.FPCA == '1' then
        framesize = 0x68;
        forcealign = '1';
    else
        framesize = 0x20;
        forcealign = CCR.STKALIGN;

    spmask = NOT(ZeroExtend(forcealign:'00',32));

    if CONTROL.SPSEL == '1' && CurrentMode == Mode_Thread then
        frameptralign = SP_process<2> AND forcealign;
        SP_process = (SP_process - framesize) AND spmask;
        frameptr = SP_process;
    else
        frameptralign = SP_main<2> AND forcealign;
        SP_main = (SP_main - framesize) AND spmask;
        frameptr = SP_main;

    /* only the stack locations, not the store order, are architected */
    MemA[frameptr,4] = R[0];
    MemA[frameptr+0x4,4] = R[1];
    MemA[frameptr+0x8,4] = R[2];
    MemA[frameptr+0xC,4] = R[3];
    MemA[frameptr+0x10,4] = R[12];
    MemA[frameptr+0x14,4] = LR;
    MemA[frameptr+0x18,4] = ReturnAddress(ExceptionType);
    MemA[frameptr+0x1C,4] = (XPSR<31:10>:frameptralign:XPSR<8:0>);
    // see ReturnAddress() in-line note for information on XPSR.IT bits
    if HaveFPExt() && CONTROL.FPCA == '1' then
        if FPCCR.LSPEN == '0' then
            CheckVFPEnabled();
            for i = 0 to 15
                MemA[frameptr+0x20+(4*i),4] = S[i];
            MemA[frameptr+0x60,4] = FPSCR;
            for i = 0 to 15
                S[i] = bits(32) UNKNOWN;
            FPSCR = bits(32) UNKNOWN;
        else
            UpdateFPCCR(frameptr);
    if HaveFPExt() then
        if CurrentMode==Mode_Handler then
            LR = Ones(27):NOT(CONTROL.FPCA):'0001';
        else
            LR = Ones(27):NOT(CONTROL.FPCA):'1':CONTROL.SPSEL:'01';
    else
        if CurrentMode==Mode_Handler then
            LR = Ones(28):'0001';
        else
            LR = Ones(29):CONTROL.SPSEL:'01';
    return;

// ExceptionTaken()
// ================
ExceptionTaken(integer ExceptionNumber)
    bit tbit;
    bits(32) tmp;
    for i = 0 to 3
        R[i] = bits(32) UNKNOWN;
    R[12] = bits(32) UNKNOWN;
    bits(32) VectorTable = VTOR<31:7>:'0000000';
    tmp = MemA[VectorTable+4*ExceptionNumber,4];
    BranchTo(tmp AND 0xFFFFFFFE<31:0>);
    tbit = tmp<0>;
    CurrentMode = Mode_Handler;
    APSR = bits(32) UNKNOWN; // Flags UNPREDICTABLE due to other activations
    IPSR<8:0> = ExceptionNumber<8:0>; // ExceptionNumber set in IPSR
    EPSR.T = tbit; // T-bit set from vector
    EPSR.IT<7:0> = Zeros(8); // IT/ICI bits cleared
    //* PRIMASK, FAULTMASK, BASEPRI unchanged on exception entry*//
    CONTROL.FPCA = '0'; // Mark Floating-point inactive
    CONTROL.SPSEL = '0'; // current Stack is Main, CONTROL.nPRIV unchanged
    //* CONTROL.nPRIV unchanged *//
    ExceptionActive[ExceptionNumber]= '1';
    SCS_UpdateStatusRegs(); // update SCS registers as appropriate
    ClearExclusiveLocal(ProcessorID());
    SetEventRegister(); // see WFE instruction for more details
    InstructionSynchronizationBarrier('1111');

// ReturnAddress()
// ===============
bits(32) ReturnAddress(integer ExceptionType)
    // xPSR.IT bits saved to the stack are consistent with ReturnAddress()
    if ExceptionType == NMI then result = NextInstrAddr();
    elsif ExceptionType == HardFault then
        result = if IsExceptionSynchronous() then ThisInstrAddr() else NextInstrAddr();
    elsif ExceptionType == MemManage then result = ThisInstrAddr();
    elsif ExceptionType == BusFault then
        result = if IsExceptionSynchronous() then ThisInstrAddr() else NextInstrAddr();
    elsif ExceptionType == UsageFault then result = ThisInstrAddr();
    elsif ExceptionType == SVCall then result = NextInstrAddr();
    elsif ExceptionType == DebugMonitor then
        result = if IsExceptionSynchronous() then ThisInstrAddr() else NextInstrAddr();
    elsif ExceptionType == PendSV then result = NextInstrAddr();
    elsif ExceptionType == SysTick then result = NextInstrAddr();
    elsif ExceptionType >= 16 then // External interrupt
        result = NextInstrAddr();
    else
        assert(FALSE); // Unknown exception number
    return result;

// UpdateFPCCR()
// =============
UpdateFPCCR(bits(32) frameptr)
    // FPCAR and FPCCR remain unmodified if CONTROL.FPCA and
    // FPCCR.LSPEN are not both set to 1
    if (CONTROL.FPCA == '1' && FPCCR.LSPEN == '1') then
        FPCAR.ADDRESS = (frameptr + 0x20)<31:3>;
        FPCCR.LSPACT = '1';
        if CurrentModeIsPrivileged() then
            FPCCR.USER = '0';
        else
            FPCCR.USER = '1';
        if CurrentMode == Mode_Thread then
            FPCCR.THREAD = '1';
        else
            FPCCR.THREAD = '0';
        if ExecutionPriority() > -1 then
            FPCCR.HFRDY = '1';
        else
            FPCCR.HFRDY = '0';
        if SHCSR.BUSFAULTENA == '1' && ExecutionPriority() > UInt(SHPR1.PRI_5) then
            FPCCR.BFRDY = '1';
        else
            FPCCR.BFRDY = '0';
        if SHCSR.MEMFAULTENA == '1' && ExecutionPriority() > UInt(SHPR1.PRI_4) then
            FPCCR.MMRDY = '1';
        else
            FPCCR.MMRDY = '0';
        if DEMCR.MON_EN == '1' && ExecutionPriority() > UInt(SHPR3.PRI_12) then
            FPCCR.MONRDY = '1';
        else
            FPCCR.MONRDY = '0';
    return;


```
