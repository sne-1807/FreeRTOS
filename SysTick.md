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

### What are the '#if' and '#endif' blocks
This concept was pretty new to me so here is what I understood about what they are. 
- `#if` and `#endif` is like a conditional compilation block (how i understood it) or formally called a  *preprocessor directive* (very fancy word, yes)
- It basically means that whatever is within this block will only be *compiled* if the condition at the `#if` block is satisfied

In our case, if `INCLUDE_xTaskGetSchedulerState == 1` condition is satisfied, the block of code will compile, which in turn is defined here:
```c
//Set the following definitions to 1 to include the API function, or zero to exclude the API functions:
#define INCLUDE_xTaskGetSchedulerState       1
```
- So we are telling the compiler:  "Hey, I have set this RTOS API to 1, so I am using it!"
- There are other RTOS APIs here that have also been assigned 0/1 but I have not included them above to avoid confusion  
  

### What is xTaskGetSchedulerState() Function

- This function is useful in specific situations where you need to know the state of the FreeRTOS scheduler
- The scheduler can be in three states and this function will return that particlar state:
  1. Running: returns  `taskSCHEDULER_RUNNING`
  2. Suspended: returns `taskSCHEDULER_SUSPENDED`
  3. Not Started: returns `taskSCHEDULER_NOT_STARTED`
```c
#if ( ( INCLUDE_xTaskGetSchedulerState == 1 ) || ( configUSE_TIMERS == 1 ) )

	BaseType_t xTaskGetSchedulerState( void )
	{
	BaseType_t xReturn;

		if( xSchedulerRunning == pdFALSE )
		{
			xReturn = taskSCHEDULER_NOT_STARTED;
		}
		else
		{
			if( uxSchedulerSuspended == ( UBaseType_t ) pdFALSE )
			{
				xReturn = taskSCHEDULER_RUNNING;
			}
			else
			{
				xReturn = taskSCHEDULER_SUSPENDED;
			}
		}

		return xReturn;
	}

#endif /* ( ( INCLUDE_xTaskGetSchedulerState == 1 ) || ( configUSE_TIMERS == 1 ) ) */
/*-----------------------------------------------------------*/
```
`configUSE_TIMERS`: if this macro is set to 1, freeRTOS software timers are enabled
- Hence this xTaskGetSchedulerState() function is only compiled and called when the `#if` conditions are satisfied

#### Why do I need a Software Timer when I already have SysTick (hardware timer)?
  
- The SysTick Timer generates interrupts at regular intervals. It's like the foundation timing system. When the SysTick interrupt occurs, it doesn't know what to do specifically
- Software Timers are fully managed by the FreeRTOS kernel, and allows you to schedule any **callback functions** that need to be called once that timer expires
- Software Timers/FreeRTOS use SysTick as it's base

**So when a SysTick Interrupt happens:**
1. FreeRTOS checks its list of software timers to see if any software timers have "expired" or if their scheduled time has arrived
2. If a software timer has expired, FreeRTOS calls the **callback function** associated with that timer

Remember that software timers donnot actually set the time limit for tasks to run, that is taken care of by the Scheduler. These are for other functions, like every 2ms read from a sensor etc (it can also be used in very FreeRTOS specific work too)

#### BaseType_t
- BaseType_t is a **typedef**, or a way to nickname an existing variable like int or long. It is not a macro.
- Since for different systems datatypes have different sizes, we use this to generalise the program
```c
#ifdef __32BIT_ARCHITECTURE__ // Example condition
typedef long BaseType_t;
#else
typedef int BaseType_t;
#endif
```
- If 32 bit system, then BaseType_t replaces long 
- It's int return values are normally used to express boolean values like 0 = FALSE and 1 = SysTick_VAL_CURRENT_Msk

Hence the return type of `xTaskGetSchedulerState` is BaseType_t which would have been defined as int/long earlier (normally a 0 or 1)

This is in the form of xReturn, which is of datatype BaseType_t and whatever gets stored in this (taskSCHEDULER_NOT_STARTED, taskSCHEDULER_RUNNING, taskSCHEDULER_SUSPENDED) get's returned when the xTaskGetSchedulerState() function is called 

```c
/* Definitions returned by xTaskGetSchedulerState().  taskSCHEDULER_SUSPENDED is
0 to generate more optimal code when configASSERT() is defined as the constant
is used in assert() statements. */
#define taskSCHEDULER_SUSPENDED		( ( BaseType_t ) 0 )
#define taskSCHEDULER_NOT_STARTED	( ( BaseType_t ) 1 )
#define taskSCHEDULER_RUNNING		( ( BaseType_t ) 2 )
```

#### Scheduler Not Running
```c
if( xSchedulerRunning == pdFALSE )
	{
			xReturn = taskSCHEDULER_NOT_STARTED;
  }
```
- `pdFALSE` is a RTOS defined macro that represents the boolean value False (maybe it's a 0)
- `xSchedulerRunning` is a normal variable acting as a flag that is default set to pdFALSE at starting. It indicates if scheduler is running, if not running, return taskSCHEDULER_NOT_STARTED
  
#### Scheduler Running
```c
if( uxSchedulerSuspended == ( UBaseType_t ) pdFALSE )
			{
				xReturn = taskSCHEDULER_RUNNING;
      }
```
- In a similar way there is a flag to check if scheduler is suspended, and returns the corresponding output when the xTaskGetSchedulerState function is called  

#### Scheduler not Started
If above two conditions are not satisfied then obviously the scheduler has not started so returns that output.

#### For Reference (How the flags used here have been initialised)
```c
PRIVILEGED_DATA static volatile BaseType_t xSchedulerRunning 		= pdFALSE;
PRIVILEGED_DATA static volatile UBaseType_t uxSchedulerSuspended	= ( UBaseType_t ) pdFALSE;
```

### Coming Back to SysTick_Handler() Function:
```c
#if (INCLUDE_xTaskGetSchedulerState == 1 )
  if (xTaskGetSchedulerState() != taskSCHEDULER_NOT_STARTED)
  {
#endif /* INCLUDE_xTaskGetSchedulerState */
  xPortSysTickHandler();
```  

- If the FreeRTOS scheduler has been started (the if condition), then the FreeRTOS handler (xPortSysTickHandler()) is called to do RTOS-related work.
- If the FreeRTOS scheduler has not been started, the xPortSysTickHandler() is not called.

### What is xPortSysTickHandler() Function:
```c
void xPortSysTickHandler( void )
{
	/* The SysTick runs at the lowest interrupt priority, so when this interrupt
	executes all interrupts must be unmasked.  There is therefore no need to
	save and then restore the interrupt mask value as its value is already
	known. */
	portDISABLE_INTERRUPTS();
	{
		/* Increment the RTOS tick. */
		if( xTaskIncrementTick() != pdFALSE )
		{
			/* A context switch is required.  Context switching is performed in
			the PendSV interrupt.  Pend the PendSV interrupt. */
			portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
		}
	}
	portENABLE_INTERRUPTS();
}
```
EXPLANATION TO BE COMPLETED 

### To summarise:
SysTick_Handler() is used mainly to increment the HAL Tick counter always, and call the xPortSysTickHandler(), which is the function where RTOS does its work like context switching, timer management, WHEN the scheduler is either running/suspended

**Doubts:** Getting confused with all the different timers(HAL increment timer, software timers by RTOS etc)
