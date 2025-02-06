# Round Robin Scheduling Code Explanation

This code is given in *Chapter 17: Multitasking (page 609)* of the textbook **Embedded Systems with ARM Cortex-M Microcontrollers in Assembly Language and C by Dr. Yifeng Zhu** 




## Introduction
* Let us assume a scenario where we have 2 tasks to choose from, task_1 and task_2, where both tasks increment a separate counter variable respectively

* While the processor is not implementing task_1 or task_2, instead of remaining idle, within the main function's while(1), we will have a default counter incrementing.

* Although this would not be considered a separate task on it's own, it would still require it's own stack frame 

## Stack Frame Data Structure
* In order to push/pop registers of any process during stacking/unstacking, we need some sort of data structure that allows us to do so by providing a layer of abstraction to make our lives easy
* Using pointers, it is very straightforward to access any register we would like by using the `->` operator(more explained later on this)

#### Question that Occured Here:
1.  *Are the variables r0, r1 etc that we are defining here in our custom structure, just variables stored somewhere in the RAM, or are they actual hardware registers? If they are just variables, how does it "connect" with actual hardware registers during pushing/popping?*

* **Answer:** 
    * The variables defined in the given stack_frame_t structure are indeed **variables** allocated some memory address within the RAM and each task would have it's own stack_frane_t structure (which is nothing but a stack). These are NOT the actual hardware registers.
    * During pushing and popping, two particularly  cool ARM instructions in your Systick_Handler() ISR that can push and pop multiple registers _in one go_ between your stack and the hardware registers called STRMB and LDMFD 


```c
typedef struct stack_frame_t
{
    //Manually pushed during stacking
   uint32_t r4; //16th item
   uint32_t r5; //15th item
   uint32_t r6;
   uint32_t r7;
   uint32_t r8;
   uint32_t r9;
   uint32_t r10;
   uint32_t r11;

   //Pushed automatically during stacking
   uint32_t r0;
   uint32_t r1;
   uint32_t r2;
   uint32_t r3;
   uint32_t r12;
   uint32_t lr;
   uint32_t pc;
   uint32_t psr; //1st item
};
```
* Creating a custom datatype with these particular variables representing all the registers we will be dealing with during context switching in it, being a structure named _stack_frame_t_ which is essentially a stack

* **Why do I need this**: 
    * _stack_frame_t_ structure is designed to provide a convenient way to access the values of registers that have been pushed/popped onto the stack by ARM instructions from hardware registers during context switching
    * So if I had 3 tasks then I would need one of these stack_frame_t structures for each of the tasks.
    * Hence, I would create three of these structures for processes 1,2,3 with this datatype. During context switching that corresponding set of registers can be stored on it's own stack

* Note: uint32_t datatype is used frequently in many places to come too, because these variables represent registers, and registers are 32 bit

## Task Table Creation
* During context switching, the stack pointer position of the previous task (be it MSP or PSP) is stored in the task table and the next task's stack pointer address is retrieved from the same table.
* Each task will have a memory address corresponding to it's stack pointer's position when context switching happens. It's essentially a mapping between the task and, to put it crudely, it's last saved stack pointer position, so the process knows where to pick up where it left off

### Structure of the Task Table
```c
typedef struct task_table_t
{
    uint32_t sp; //stack pointer
    int flags;
};
```
### Allocate the Task Table and Track Running Task
```c
task_table_t task_table[3];
//task_table[0]: stack of main task 
//task_table[1]: stack of task 1 
//task_table[2]: stack of task 2

//Keep track of which task is running
int current_task =0;
```

* Global task table to record stack pointer and status flags for each task
* **Important**: MSP used for main task and PSP used for task 1,2 

## Accessing Variables in Stack (Welcoming Our Friend: Pointers)
* Accessing each stack's registers using the _task_table_ stack pointer _sp_ and casting it to stack frame pointer to access the content of a register stored in the stack directly

```c
stack_frame_t* frame;
frame = (stack_frame_t *)(task_table[i].sp-sizeof(stack_frame_t)); 
```
* **Explanation**: 
    1. We defined a stack frame pointer `stack_frame_t *frame`
        * Just the way an integer pointer points to an integer held in some memory location, the stack frame pointer called `frame` will point to a _stack_frame_t_ data structure held in RAM location. 
    2. We _cast_ our stack pointer (sp) to a stack frame pointer (frame)
        * `task_table[i].sp - sizeof(stack_frame_t);`: calculates the memory address where the stack_frame_t structure for this task begins. Remember, the stack grows downwards in memory. 
        * Since ```task_table[i].sp - sizeof(stack_frame_t);``` is probably stored as a number (memory address), you cannot assign a pointer of the datatype _stack_frame_t_ to it directly. Hence we do "casting" to tell the compiler that the number represents the address of a stack_frame_t structure which is being pointed to by a stack_frame_t pointer called `frame`.
        * It sounds a bit redundant, you could be wondering why we need to explicitly tell the compiler again that `frame` is a stack_frame_t pointer, but you just have to do it. Note that we use the * symbol to indicate that it is a pointer and not your regular ol' datatype.
    * The `frame` pointer will point to the top current task's stack frame (so it will either point to starting of task 1, 2 or 3's stack frame, indicating which stack is currently in use)
    * During context switching the `frame` pointer will point to the newly selected task
    
```c
frame-> r0 = 0;
frame-> pc = ((unit32_t)func) 
```
* **Explanation**
    * `-> (arrow operator)`: Used to access members of a structure when you have a **pointer** to that structure ( . operator used when you have a variable of that structure e.g. task_table[i].sp )
    * In `frame-> r0 = 0;`, we set r0 register of that stack as 0 (this register will be used frequently in the context of MSP and PSP as you will see later)
    * `frame-> pc = ((uint32_t)func)`
        * We are accessing the PC of the selected tasks's stack frame.
        * _Brush Up_:  PC stores the memory address of the next instruction to be executed, so the compiler knows where to go next.
        * Let's say we are choosing task 1. The PC should store the memory address of the first line of the task 1 code block. 

        * `func`: is a function pointer, holding the memory address of the task's starting point\first instruction of a function.

            **What the hell is func:** 
            What happens when you call a function called, say, hello() in your main code? The compiler will go to the part of code where you have defined hello() and execute the whole thing. However if your main code contains only hello; then it will go to the hello() function and only execute it's first line. Same idea here!
        
        * Note that `func` is a place holder technically. The same way our r0-r14etc registers are "placeholders" in the stack_frame structure that get loaded into the hardware registers same goes with this `func`. `func` would get replaced by the name of the actual task when you call the function `new_task` (coming later on in the file)  

        * `uint32_t`: is for casting whatever that function pointer holds as 32 bit integer as that is what the register pc can store
        
```c
task_table[i].sp=((uint32_t)frame)
task_table[i].flags = TASK_FLAG_EXEC | TASK_FLAG_INIT; 
```
* **Explanation**
    * `task_table[i].sp=((uint32_t)frame)`: the `frame` pointer points to the starting address of the stack frame of the task selected. We defined that in the first two lines of code, and now we are actually assigning it to the stack pointer, which goes to the correct stack for execution of that task to access/unstack the correct registers
    * **(???????????)okay so here is my question. if sp needs to be be popping values out of the selected task's stack when context switching happens, and frame points to the place where the stack starts, how does that make sense? the stack grows downwards, so shouldn't it be pointing to the bottom most/last pushed register's memory address if it needs to pop that?**
    * `task_table[i].flags = TASK_FLAG_EXEC | TASK_FLAG_INIT;`  sets the flags member for the i-th task in the task_table to a combination of TASK_FLAG_EXEC and TASK_FLAG_INIT. This signifies that the task has been initialized and is ready to be executed by the scheduler.

## Main Code
```c
int main(void)
{ 
    uint32_t k, var = 1; 
    tasks_init(); 
    new_task(my_task_l, (uint32_t)&var); 
    new_task(my_task_2, (uint32_t)&var);
```
* **Explanation:**
    * `uint32_t k, var=1` initialising k for main task's delay between incrementing counter 
    * var (????)
    * (????) tasks_init() 
    * `new_task(my_task_1, (uint32_t)&var)` used to initialise a new task
        * `my_task_1` argument is a *function pointer* that points to the first line of code of the actual function (as you will see under the explanation for the `new_task` function definition coming up later on)
        * `(uint32_t)&var` argument takes memory address of `var` variable and converts it to a 32 bit unsigned int (????????? what is var for!)
```c
    SysTick_Init(); //Initialise the SysTick Timer 
    NVIC_EnableIRQ(SysTick_IRQn); // Enable SysTick interrupt in NVIC
    while(l)
    { 
        counter_main++; 
        for(k = 0; k < 1000; k++); 
    }
} 
```
* **Explanation:**
    * `NVIC_EnableIRQ(SysTick_IRQn)`
    * **Nested Vector Interrupt Controller (NVIC)**: is used to manage all interrupts. This line of code enables the SysTick interrupt in the NVIC, basically telling the processor to respond to any SysTick Interrupt
    * Without this line of code:
        * The SysTick timer would continue to count down and generate interrupt requests, but the processor would ignore these requests.
        * The SysTick_Handler() function would never be called.
    * Now if a SysTick Interrupt happens, it will automatically jump the SysTick_Handler() (the ISR routine) to do the actual context switching etc



## Reading and Writing to MSP and PSP
### Some Concepts about MSP/PSP:
There is always one stack pointer, called `sp`, and it is register r13
in any ARM processor. This `sp` can behave as either a `Main Stack Pointer (MSP)` or a `Process Stack Pointer (PSP)` depending on whether it is accessing the OS's stack or a process' stack respectively. 

When operating in _privileged_ mode, like during exception handling, which is done frequently in this round robin code (e.g. Systick_Handler()), certain special registers need to be accessed which cannot be done in _unprivieleged mode_ (which is what the task's will run in). Hence in this case **MSP** is used for _privileged mode_. Since the main function has a bunch of privilege mode functions like Systick_Init() etc, we need to use MSP for the main function, **and as a result, the task within the while(1) loop of main function also uses MSP (????? does it or is main stack only for interrupt handling)**

When the tasks are running (task_1 and task_2) they use PSP to point to their respective process stacks. Note that since the tasks are runnning in unpriveleged mode, they will use the PSP. 

Essentially since there will be a dedicated **Main Stack** and  **Process Stack(s)** depending on number of tasks (n tasks = n process stacks). The Main Stack always uses MSP and Process Stack always uses PSP. You could use one stack pointer for all the stacks but it becomes a lot more complicated and issue-prone. Hence we have two stack pointers for 2 types of stacks (always one main stack and can have more that one process stack, but all those process stacks will use PSP).

### Flow of Events (related to SP) during context switching

* Pending!!!!!!!!!!

### MSP and PSP functions
* `__asm` indicates function is written in assembly
* `get_MSP()` and `get_PSP()` are used when starting the _next_ task after context switching, to retrieve the address of the MSP/PSP before that task was halted earlier (it retrieves the address that MSP/PSP needs to be pointing to from r0)
* `set_MSP()` and `set_PSP()` are used for the task that get's stopped during context switching. The current MSP/PSP values are loaded onto r0, which will be pushed onto stack and accessed later during context switching when this task is called again.

* **(????????????) in both the set functions there is an argument being used  but it is not being used anywhere in the actual function code?** 
* **r0(??????)**
#### MSP Functions
```c
__asm uint32_t get_MSP(void) //Read main stack pointer
{
    MRS r0, msp; //Copy msp to r0 
    BX lr;
}
```

```c
__asm void set_MSP(uint32_t topStackPointer) //Write to main stack pointer
{
    MSR msp, r0; //Copy r0 to msp
    BX lr;
}
```

#### PSP Functions
```c
__asm uint32_t get_PSP(void) //Read from PSP
{
    MRS r0, psp; //Copy psp to r0
    BX lr;
}
```

```c
__asm void set_PSP(uint32_t topStackPointer) //Write to PSP
{
    MSR psp, r0; //Copy r0 to psp
    BX lr;
}
```

## New_Task Create Function:

```c
void new_task(void* func(void*), uint32_t args)
{
    int i; 
    stack_frame_t * frame; 
    for(i=1; i < MAX_TASKS; i++)
    {
        if(task_table[i].flags == 0)
        {
            frame= (stack_frame_t *)( task_table[i].sp - sizeof(stack_frame_t) );
            frame->r4 = 0; 
            frame->r5 = 0; 
            frame->r6 = 0; 
            frame->r7 = 0;
            frame->r8 = 0;
            frame->r9 = 0;
            frame->r10 = 0;
            frame->r11 = 0;

            frame->r0 = ((uint32_t)args);
            frame->r1= 0;
            frame->r2= 0;
            frame->r3 = 0;
            frame->r12 = 0;
            frame->pc = ((uint32_t)func);
            frame->lr = 0;
            frame->psr = 0x21000000;

            task_table[i].flags = TASKS_FLAG_EXEC | TASKS_FLAG_INIT;
            task_table[i].sp = ((uint32_t)frame);
            set_PSP(task_table[i].sp); 
            break;
        }
    }
} 
 
 
 
 
 
 
 
 
 


    }