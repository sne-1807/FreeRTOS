# SysTick Theory
## Introduction
*  24-bit down counter
*  Generates periodic interrupts to execute a task repeatedly.
*  Processor generates a SysTick interrupt once the counter reaches zero.
*  SysTick counter loads the value held in a special register named the SysTick Reload register and counts down again. 

## Application in RTOS
* SysTick acts as a hardware timer for the CPU scheduler in real-time operating systems (RTOS).
* When multiple tasks run concurrently, the processor allocates a time slot to each task according to some scheduling policy, such 
as the round robin. To achieve that, the processor utilizes a hardware timer to generate 
interrupts at regular time intervals. These interrupts inform the processor to stop the 
current task, save the context registers of the present task to the stack, and then select a 
new task in the job waiting queue to serve.

## Systick Registers

* There are four 32-bit registers for configuring system timers
  
### 1. SysTick_CTRL
**Bit Pos 0: ENABLE**  
`0` = Counter disabled 
`1` = Counter enabled 

**Bit Pos 1: TICKINT**   
TICKINT enables SysTick interrupt request:    
`0` =Counting down to zero does not assert the SysTick interrupt request   
`1` =Counting down to zero asserts the SysTick interrupt request 

**Bit Pos 2: CLKSOURCE**  
`0` = External clock. The frequency of SysTick clock is the frequency of the AHB clock divided by 8.   
`1` = Processor clock 

**Bit Pos 16: COUNTFLAG**  
Indicates whether a special event has occurred.  
`0`= Counter has transitioned from 1 to 0 since the last read of SysTick_CTRL   
`1`=  COUNTFLAG is cleared by reading SysTick_CTRL or by writing to SysTick_VAL.

(Rest of the bit positions are reserved)

## 2. SysTick_LOAD

* After *the counter counts down to zero, the counter restarts from the value in `SysTick_LOAD`*
* **If the SysTick interrupt is required once every N clock pulses, software should set SysTick_LOAD to N - 1**

## 3. SysTick_VAL
* When SysTick is enabled, the 24-bit current counter value in the SysTick_VAL 
register is copied from the SysTick_LOAD register initially
* The processor automatically decrements SysTick_VAL by one at each clock pulse 
sent to the timer
* **`SysTick_VAL` is like a digital display on a countdown timer that shows you the current value of the timer. You can read it to check the remaining time or reset the time**
* Writing to SysTick_VAL register also **clears the COUNTERFLAG** in the SysTick_CTRL register

## 4. SysTick_CALIB
* **SysTick Calibration Value Register contains a value that represents the number of ticks required for a 10ms interval**
* This value can be used to calculate the reload value for the SysTick Reload Value Register, which determines the frequency of the SysTick interrupts
* For example, if the SysTick Calibration Value Register contains the value 1000, this means that it takes 1000 ticks for the SysTick timer to count down from its maximum value to 0
* The Calibration Value Register is a tool to help you achieve accurate timing with the SysTick timer, especially when you need precise intervals or are dealing with potential clock speed variations.
* Not necessary to use it

## SysTick Interrupt Period Formula
*Systick Interrupt Period = (1+SysTick_LOAD) **x** (1/SysTick Counter Clock Frequency)*



# SysTick Implementation using HAL (STM32CubeMX)
Below, I have extracted the code related to SysTick only from the entire generated code by CubeMX.

**DISCALIMER: PLEASE NOTE THAT I HAVE TAKEN THE RELEVANT BITS OF CODE ASSOCIATED WITH SYSTICK ONLY**

## SysTick Related #define Statements
### Volatile Keyword for SysTick's 4 Registers
* SysTick timer is a hardware counter,  hence all the registers associated with it are hardware registers. In order to prevent the compiler from performin any optimisation/hardcoding, `volatile` keyword is used
* This ensures that data read/write will happen in the **memory** location of the hardware register
* When running code, the memory location's data is placed on a CPU register, which makes it faster for it to access. If you don't declare it as volatile, the compiler will assume that, since the value is not changing, **it will keep reading the value from the CPU register (and that value may be outdated) instead of the checking for the last updated value in the RAM memory**

```c
#define     __IM     volatile const      /*! Defines 'read only' structure member permissions */
#define     __OM     volatile            /*! Defines 'write only' structure member permissions */
#define     __IOM    volatile            /*! Defines 'read / write' structure member permissions */
```

### Base Address of SysTick Peripheral/Counter
```c
#define SCS_BASE            (0xE000E000UL)                            /*!< System Control Space Base Address */

#define SysTick_BASE        (SCS_BASE +  0x0010UL)                    /*!< SysTick Base Address */
```

### Other Definitions That May Help Later
```c
 #define  __STATIC_INLINE             static inline
```
```c
/* SysTick Control / Status Register Definitions */
#define SysTick_CTRL_COUNTFLAG_Pos         16U                                            /*!< SysTick CTRL: COUNTFLAG Position */
#define SysTick_CTRL_COUNTFLAG_Msk         (1UL << SysTick_CTRL_COUNTFLAG_Pos)            /*!< SysTick CTRL: COUNTFLAG Mask */

#define SysTick_CTRL_CLKSOURCE_Pos          2U                                            /*!< SysTick CTRL: CLKSOURCE Position */
#define SysTick_CTRL_CLKSOURCE_Msk         (1UL << SysTick_CTRL_CLKSOURCE_Pos)            /*!< SysTick CTRL: CLKSOURCE Mask */

#define SysTick_CTRL_TICKINT_Pos            1U                                            /*!< SysTick CTRL: TICKINT Position */
#define SysTick_CTRL_TICKINT_Msk           (1UL << SysTick_CTRL_TICKINT_Pos)              /*!< SysTick CTRL: TICKINT Mask */

#define SysTick_CTRL_ENABLE_Pos             0U                                            /*!< SysTick CTRL: ENABLE Position */
#define SysTick_CTRL_ENABLE_Msk            (1UL /*<< SysTick_CTRL_ENABLE_Pos*/)           /*!< SysTick CTRL: ENABLE Mask */

/* SysTick Reload Register Definitions */
#define SysTick_LOAD_RELOAD_Pos             0U                                            /*!< SysTick LOAD: RELOAD Position */
#define SysTick_LOAD_RELOAD_Msk            (0xFFFFFFUL /*<< SysTick_LOAD_RELOAD_Pos*/)    /*!< SysTick LOAD: RELOAD Mask */

/* SysTick Current Register Definitions */
#define SysTick_VAL_CURRENT_Pos             0U                                            /*!< SysTick VAL: CURRENT Position */
#define SysTick_VAL_CURRENT_Msk            (0xFFFFFFUL /*<< SysTick_VAL_CURRENT_Pos*/)    /*!< SysTick VAL: CURRENT Mask */

/* SysTick Calibration Register Definitions */
#define SysTick_CALIB_NOREF_Pos            31U                                            /*!< SysTick CALIB: NOREF Position */
#define SysTick_CALIB_NOREF_Msk            (1UL << SysTick_CALIB_NOREF_Pos)               /*!< SysTick CALIB: NOREF Mask */

#define SysTick_CALIB_SKEW_Pos             30U                                            /*!< SysTick CALIB: SKEW Position */
#define SysTick_CALIB_SKEW_Msk             (1UL << SysTick_CALIB_SKEW_Pos)                /*!< SysTick CALIB: SKEW Mask */

#define SysTick_CALIB_TENMS_Pos             0U                                            /*!< SysTick CALIB: TENMS Position */
#define SysTick_CALIB_TENMS_Msk            (0xFFFFFFUL /*<< SysTick_CALIB_TENMS_Pos*/)    /*!< SysTick CALIB: TENMS Mask */

/*@} end of group CMSIS_SysTick */


```

## SysTick_Type Structure
* Made `CTRL, LOAD, VAL, CALIB` all volatile but only `CTRL, LOAD and VAL` are both read/write (i.e. we can change them) but `CALIB` is read only
* The SysTick_CALIB register is read-only because its value is determined by the hardware and is not intended to be modified by the user.  It's a calibration value, set during manufacturing or board initialization, and reflects the characteristics of the specific microcontroller.

```c
typedef struct
{
  __IOM uint32_t CTRL;                   /*!< Offset: 0x000 (R/W)  SysTick Control and Status Register */
  __IOM uint32_t LOAD;                   /*!< Offset: 0x004 (R/W)  SysTick Reload Value Register */
  __IOM uint32_t VAL;                    /*!< Offset: 0x008 (R/W)  SysTick Current Value Register */
  __IM  uint32_t CALIB;                  /*!< Offset: 0x00C (R/ )  SysTick Calibration Register */
} SysTick_Type;
```

### Offsets and Base Addresses
* Hardware peripherals like the SysTick timer are accessed through memory-mapped registers. This means that each register is assigned a specific memory address. To interact with a register, you need to read from or write to its corresponding memory location

* Every peripheral has a base address, which is the starting address of its memory space.   
* The individual registers within the peripheral are located at specific offsets from this base address.   

#### In our case:
Each register is 32 bits = 4 bytes and should have a difference of 4 bytes memory space between one another

1. `CTRL` is at offset 0x000, meaning it's located at the base address of the SysTick peripheral.
2. `LOAD` is at offset 0x004, meaning it's located 4 bytes away from the base address.
3. `VAL` is at offset 0x008, meaning it's 8 bytes away from the base address.
4. `CALIB` is at offset 0x00C, meaning it's located at 12 bytes away from the base address of  SysTick peripheral.

**NOTE:** Offset is in comments for our understanding of the memory mapping of these hardware registers to RAM memory

## SysTick Pointer Definition

```c
#define SysTick             ((SysTick_Type   *)     SysTick_BASE  )   /*!< SysTick configuration struct */
```
* Defining a pointer called `SysTick`, of the datatype of the `SysTick_Type` structure, pointing to the base address of the SysTick Peripheral
* We can increment this pointer to point to different registers. Since the size of the `SysTick_Type` structure is 4 bytes, incrementing/decrementing the pointer by 1 will make it jump 4 bytes at a time, pointing to one of the 4 registers.

## Systick Configuration: SysTick_Config()
The `#if defined (...)` block means this code is only included if __Vendor_SysTickConfig is not defined or is set to 0.  If it's set to 1, it's assumed the vendor provides their own implementation

```c
/* ##################################    SysTick function  ############################################ */
/*
  \ingroup  CMSIS_Core_FunctionInterface
  \defgroup CMSIS_Core_SysTickFunctions SysTick Functions
  \brief    Functions that configure the System.
  @{
 */

#if defined (__Vendor_SysTickConfig) && (__Vendor_SysTickConfig == 0U)

/*
  \brief   System Tick Configuration
  \details Initializes the System Timer and its interrupt, and starts the System Tick Timer.
           Counter is in free running mode to generate periodic interrupts.
  \param [in]  ticks  Number of ticks between two interrupts.
  \return          0  Function succeeded.
  \return          1  Function failed.
  \note    When the variable <b>__Vendor_SysTickConfig</b> is set to 1, then the
           function <b>SysTick_Config</b> is not included. In this case, the file <b><i>device</i>.h</b>
           must contain a vendor-specific implementation of this function.
 */
__STATIC_INLINE uint32_t SysTick_Config(uint32_t ticks)
{
  if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk)
  {
    return (1UL);                                                   /* Reload value impossible */
  }

  SysTick->LOAD  = (uint32_t)(ticks - 1UL);                         /* set reload register */
  NVIC_SetPriority (SysTick_IRQn, (1UL << __NVIC_PRIO_BITS) - 1UL); /* set Priority for Systick Interrupt */
  SysTick->VAL   = 0UL;                                             /* Load the SysTick Counter Value */
  SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |
                   SysTick_CTRL_TICKINT_Msk   |
                   SysTick_CTRL_ENABLE_Msk;                         /* Enable SysTick IRQ and SysTick Timer */
  return (0UL);                                                     /* Function successful */
}

#endif

/*@} end of CMSIS_Core_SysTickFunctions */

```
### Code Explanation/Breakdown
```c
if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk)
  {
    return (1UL);                                                   /* Reload value impossible */
  }
```
**Main Goal:** Checks if the number of ticks that have been given by user is valid

*  `ticks - 1UL` denotes that if SysTick Interrupt is required once ever N clock pulses i.e. **ticks**, the value to be loaded is N-1 or ticks-1
*  `#define SysTick_LOAD_RELOAD_Msk            (0xFFFFFFUL /*<< SysTick_LOAD_RELOAD_Pos*/)` represents the maximum value that can be loaded into the SysTick_LOAD register.
*  Since SysTick is 24 bit counter: (4*6 Fs) = 24 bits denoted by 0xFFFFFF
*  Hence if your number of ticks defined exceeds that, then it is invalid

**IF NUMBER OF TICKS IS VALID THEN ENTER HERE:**

#### Load Ticks into Reload Register
```c
SysTick->LOAD  = (uint32_t)(ticks - 1UL);   
```
* Accessed the SysTick_LOAD register and loads the `ticks - 1UL` value

#### Set Least Priority for SysTick Intterupt
```c
NVIC_SetPriority (SysTick_IRQn, (1UL << __NVIC_PRIO_BITS) - 1UL);
```
* This line of code **sets the SysTick interrupt to the lowest possible priority level**.  This means that if other interrupts occur while the SysTick interrupt is being handled, those other interrupts will be able to preempt (interrupt) the SysTick handler if they have a higher priority.
* `NVIC_SetPriority` is a function to set the priority of an interrupt
* `SysTick_IRQn` is an enumeration constant that represents the interrupt number of the SysTick interrupt, which is `-1` 
* **`__NVIC_PRIO_BITS` defines the number of bits used for priority levels in the NVIC**
    * **In the CMSIS code of STM32F4xx, it was `4` bits, hence there are 2^4 = 16 priority levels starting from 0**
    * **Priority level 15 is highest priority level and Priority level 0 is the lowest priority level**
* Hence doing `(1UL << __NVIC_PRIO_BITS)` essentially does 2^__NVIC_PRIO_BITS which is 2^4 = 16 (as each left shift operations multiplies 1 with 2)
* Finally subtracting by 1, as top most priority level is 15

#### Restart Countdown and Clear Count Flag
```c
SysTick->VAL   = 0UL;
```
* Writing 0 to SysTick->VAL effectively **restarts the countdown cycle.**
* Writing to the SysTick->VAL register also has a side effect being it **clears the COUNTFLAG** bit in the SysTick->CTRL register

#### Configure Control Register and Start SysTick
```c
SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |
                   SysTick_CTRL_TICKINT_Msk   |
                   SysTick_CTRL_ENABLE_Msk;   
```
* Bitwise OR operator `|` combines the three masks together. The result is a single value where the **bits corresponding to the clock source, interrupt enable, and timer enable are all set to 1**.
* Creating 3 masks to do:
    1. Set the clock source to be the processor clock (=1) by left shifting 1 by two bits to the bit position of CLKSOURCE in the CTRL reg
    2. Set TICKINT, bitwise shifting of 1 to bit 1 position in CTRL reg (position of TICKINT) tells the SysTick timer to generate an interrupt when it counts down to zero
    3. Lastly, the enable mask enables the SysTick timer itself. Setting this bit starts the timer counting down.
**(HOW DOES IT WORK FOR ENABLE WHEN THE DEFINE FUNCTION DOES NOT SHIFT THE 1 to the BIT POSITION 15!? unlike the other 2 masks??????????????????????????????????????????????????????????)**
  
**Masks used here are (for understanding):**
```c
#define SysTick_CTRL_CLKSOURCE_Pos          2U                                           
#define SysTick_CTRL_CLKSOURCE_Msk         (1UL << SysTick_CTRL_CLKSOURCE_Pos)   

#define SysTick_CTRL_TICKINT_Pos            1U                                           
#define SysTick_CTRL_TICKINT_Msk           (1UL << SysTick_CTRL_TICKINT_Pos)             

#define SysTick_CTRL_ENABLE_Pos             0U                                           
#define SysTick_CTRL_ENABLE_Msk            (1UL /*<< SysTick_CTRL_ENABLE_Pos*/)          
```

## Systick Handler: SysTick_Handler()
```c
void SysTick_Handler(void)
{
  /* USER CODE BEGIN SysTick_IRQn 0 */

  /* USER CODE END SysTick_IRQn 0 */
  HAL_IncTick();
#if (INCLUDE_xTaskGetSchedulerState == 1 )
  if (xTaskGetSchedulerState() != taskSCHEDULER_NOT_STARTED)
  {
#endif /* INCLUDE_xTaskGetSchedulerState */
  xPortSysTickHandler();
#if (INCLUDE_xTaskGetSchedulerState == 1 )
  }
#endif /* INCLUDE_xTaskGetSchedulerState */
  /* USER CODE BEGIN SysTick_IRQn 1 */

  /* USER CODE END SysTick_IRQn 1 */
}
```

### HAL_IncTick()
```c
__weak void HAL_IncTick(void)
{
  uwTick += uwTickFreq;
}
```
* This function's job is to increment the `uwTick` variable.  If `uwTickFreq` is 1000, `uwTick` will increase by 1000 every time this function is called, effectively counting milliseconds.
* Since `uwTickFreq` is 1000 Hz, the time period is 1/1000 = 1ms. So each tick represents 1 ms and that is being added to the variable `uwTick`
* Every time a "clock tick" (SysTick interrupt) occurs, this function increments a counter (uwTick) so that your program knows how much time has passed.
#### Definitions to help understand:
```c
__IO uint32_t uwTick;
HAL_TickFreqTypeDef uwTickFreq = HAL_TICK_FREQ_DEFAULT;  /* 1KHz */
```

**I DON'T UNDERSTAND THE WAY THIS CONDITIONAL BLOCK WORKS. THE WHOLE #if, if etc (?????????????????)**

============== To be Finished :} ======================