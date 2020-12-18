# MP3_report_39

## Team Member & Contributions

 * ## **107062301 李昆諭**
 * ## **107060017 廖凰汝**

| 工作項目   | 分工            |
| ---------- | --------------- |
| Trace Code | 李昆諭 |
| 報告撰寫 | both |
| 功能實作   |  廖凰汝  |

# Trace code

用state disgram幫助理解:

![](https://github.com/107062301/jeff42105/blob/master/osmp3.jpg?raw=true)

# 1-1. New→Ready
### Kernel::ExecAll()
```javascript=
void Kernel::ExecAll()
{
    for (int i=1; i<=execfileNum; i++) {
        int a = Exec(execfile[i]);
    }  
    currentThread->Finish(); 
}
```
1.  在for loop 中呼叫 **Exec()** 來執行多個 **exefile**
2.  若所有 **Exec(exefile[i])** 結束, currentThread call **Finish()** 來結束NachOS

### Kernel::Exec(char*)
```javascript=
int Kernel::Exec(char* name)
{
    t[threadNum] = new Thread(name, threadNum);
    t[threadNum]->space = new AddrSpace(usedPhysics);//modify
    t[threadNum]->Fork((VoidFunctionPtr) &ForkExecute, (void *)t[threadNum]);
    threadNum++;

    return threadNum-1;
}
```
1.	用 array **t** 來記錄threads, 將create完的new thread用 **t[threadNum]** 儲存
2.	allocate address space for the new thread
3.	call **t->Fork()** allocate and initialize a stack

### Thread::Fork(VoidFunctionPtr, void*)
```javascript=
void 
Thread::Fork(VoidFunctionPtr func, void *arg)
{
    Interrupt *interrupt = kernel->interrupt;
    Scheduler *scheduler = kernel->scheduler;
    IntStatus oldLevel;
    
    DEBUG(dbgThread, "Forking thread: " << name << " f(a): " << (int) func << " " << arg);
    StackAllocate(func, arg);

    oldLevel = interrupt->SetLevel(IntOff);
    scheduler->ReadyToRun(this);	// ReadyToRun assumes that interrupts 
					                // are disabled!
    (void) interrupt->SetLevel(oldLevel);
}
```
1.	call **StackAllocate()** to set MachineState for the thread that just created, and the parameter **func** and **arg** are passed to **StackAllocate()** 
2.	turn off interrupt and store previous level in **oldLevel**
3.	push the thread in ready queue (**ReadyToRun(this)**)
4.	set the interrupt back to **oldLevel**

### Thread::StackAllocate(VoidFunctionPtr, void*)
* 截錄StackAllocate的關鍵程式碼
```javascript=
void
Thread::StackAllocate(VoidFunctionPtr func, void *arg)
{
        
    stack = (int *) AllocBoundedArray(StackSize * sizeof(int));

    stackTop = stack + StackSize - 4;	
        
    *(--stackTop) = (int) ThreadRoot;
    
    *stack = STACK_FENCEPOST;
       
    machineState[PCState] = (void*)ThreadRoot;
    machineState[StartupPCState] = (void*)ThreadBegin;
    machineState[InitialPCState] = (void*)func;
    machineState[InitialArgState] = (void*)arg;
    machineState[WhenDonePCState] = (void*)ThreadFinish;
}
```
1.	allocate a memory protected array to stack using **AllocBoundedArray()**
2.	根據指令集架構讓 **stackTop** 指向特定的stack位置
3.	push the address of **ThreadRoot()** onto stack **(x86)**
4.	set ***stack = STACK_FENCEPOST** for detecting stack overflow
5.	the parameters **(func, arg)** stand for **(&ForkExecute, t[threadNum])** , which will be executed when we **Run()** this thread

### Scheduler::ReadyToRun(Thread*)
```javascript=
void
Scheduler::ReadyToRun (Thread *thread)
{
    ASSERT(kernel->interrupt->getLevel() == IntOff);
    DEBUG(dbgThread, "Putting thread on ready list: " << thread->getName());
	//cout << "Putting thread on ready list: " << thread->getName() << endl ;
    thread->setStatus(READY);
    readyList->Append(thread);
}
```
1.	check whether interrupt is disabled
2.	set status of the thread to **READY**
3.	append this thread to the readyList

# 1-2. Running→Ready

### Machine::Run()
```javascript=
void Machine::Run() {
  Instruction *instr = new Instruction; // storage for decoded instruction

  if (debug->IsEnabled('m')) {
    cout << "Starting program in thread: " << kernel->currentThread->getName();
    cout << ", at time: " << kernel->stats->totalTicks << "\n";
  }
  kernel->interrupt->setStatus(UserMode);
  for (;;) {
    DEBUG(dbgTraCode, "In Machine::Run(), into OneInstruction "
                          << "== Tick " << kernel->stats->totalTicks << " ==");
    OneInstruction(instr);
    DEBUG(dbgTraCode, "In Machine::Run(), return from OneInstruction  "
                          << "== Tick " << kernel->stats->totalTicks << " ==");

    DEBUG(dbgTraCode, "In Machine::Run(), into OneTick "
                          << "== Tick " << kernel->stats->totalTicks << " ==");
    kernel->interrupt->OneTick();
    DEBUG(dbgTraCode, "In Machine::Run(), return from OneTick "
                          << "== Tick " << kernel->stats->totalTicks << " ==");
    if (singleStep && (runUntilTime <= kernel->stats->totalTicks))
      Debugger();
  }
}
```
1.	Set kernel status to user mode
2.	create a new instruction storage for decoded instruction
3.	repeatedly run **OneInstruction()** (execute an instruction from a user-level program) and **OneTick()** (simulate time)

### Interrupt::OneTick()
```javascript=
void Interrupt::OneTick() {
  MachineStatus oldStatus = status;
  Statistics *stats = kernel->stats;

  // advance simulated time
  if (status == SystemMode) {
    stats->totalTicks += SystemTick;
    stats->systemTicks += SystemTick;
  } else {
    stats->totalTicks += UserTick;
    stats->userTicks += UserTick;
  }
  DEBUG(dbgInt, "== Tick " << stats->totalTicks << " ==");

  // check any pending interrupts are now ready to fire

  ChangeLevel(IntOn, IntOff); // first, turn off interrupts
                              // (interrupt handlers run with
                              // interrupts disabled)
  CheckIfDue(FALSE);          // check for pending interrupts
  ChangeLevel(IntOff, IntOn); // re-enable interrupts
  if (yieldOnReturn) {        // if the timer device handler asked
                              // for a context switch, ok to do it now
    yieldOnReturn = FALSE;
    status = SystemMode; // yield is a kernel routine
    kernel->currentThread->Yield();
    status = oldStatus;
  }
}
```
1.	檢查目前的status (SystemMode or UserMode)
2.	根據status增加ticks,同時以 ***stats = kernel->stats** 來增加total ticks
3.	Disable interrupt, call **CheckIfDue()** 來檢查是否有pending interrupt尚未執行
4.	**CheckIfDue()** return之後, re-enable interrupt
5.	若 **yieldOnReturn == true** , 代表要切換到另一個thread, 先將 **yieldOnReturn** 設為false後再進入system mode執行 **Yield()** , 當 **Yield()** return後, 將status設為 **oldStatus**

### Thread::Yield()
```javascript=
void
Thread::Yield ()
{
    Thread *nextThread;
    IntStatus oldLevel = kernel->interrupt->SetLevel(IntOff);
    
    ASSERT(this == kernel->currentThread);
    
    DEBUG(dbgThread, "Yielding thread: " << name);
    
    nextThread = kernel->scheduler->FindNextToRun();
    if (nextThread != NULL) {
	    kernel->scheduler->ReadyToRun(this); //把舊的thread加進ready list
	    kernel->scheduler->Run(nextThread, FALSE);
    }
    (void) kernel->interrupt->SetLevel(oldLevel);
}
```
1.	Disable interrupt, **Yield()** 的過程不容許中斷
2.	Call **FindNextToRun()**, 找到要執行的thread, 用 ***nextThread** 存取該thread
3.	將current thread append進ready list(run => ready),並且執行(Run) **nextThread**
4.  After Run() return, 將interrupt level 恢復為 **oldLevel**

### Scheduler::FindNextToRun()
```javascript=
Thread *
Scheduler::FindNextToRun ()
{
    ASSERT(kernel->interrupt->getLevel() == IntOff);

    if (readyList->IsEmpty()) {
		return NULL;
    } else {
    	return readyList->RemoveFront();
    }
}
```
1.	檢查interrupt是否disabled
2.	Return 一個ready list的thread,若list為空則return NULL, 否則remove ready list的最前面一個thread並將它return


### Scheduler::ReadyToRun(Thread*)
_※和1-1的 **ReadyToRun** 一樣_
1.	檢查interrupt是否disabled
2.	將傳入的thread的status設為 **READY** 
3.	將該thread append進ready list

### Scheduler::Run(Thread*, bool)
```javascript=
void
Scheduler::Run (Thread *nextThread, bool finishing)
{
    Thread *oldThread = kernel->currentThread;
    
    ASSERT(kernel->interrupt->getLevel() == IntOff);

    if (finishing) {	// mark that we need to delete current thread
         ASSERT(toBeDestroyed == NULL);
	     toBeDestroyed = oldThread;
    }
    
    if (oldThread->space != NULL) {	// if this thread is a user program,
        oldThread->SaveUserState(); 	// save the user's CPU registers
	    oldThread->space->SaveState();
    }
    
    oldThread->CheckOverflow();		    // check if the old thread
					    // had an undetected stack overflow

    kernel->currentThread = nextThread;  // switch to the next thread
    nextThread->setStatus(RUNNING);      // nextThread is now running
    
    DEBUG(dbgThread, "Switching from: " << oldThread->getName() << " to: " << nextThread->getName());
    
    SWITCH(oldThread, nextThread);

    ASSERT(kernel->interrupt->getLevel() == IntOff);

    DEBUG(dbgThread, "Now in thread: " << oldThread->getName());

    CheckToBeDestroyed();	
	
    if (oldThread->space != NULL) {	    // if there is an address space
        oldThread->RestoreUserState();     // to restore, do it.
	    oldThread->space->RestoreState();
    }
}
```
_※這裡的finishing變數是false, 因為old thread並沒有結束，只是暫時把CPU讓給其他thread去執行(**Yield()**)_
1.	用 ***oldThread** 儲存current thread的位址
2.	check whether interrupt is disabled
3.	if the old thread is finished, 將 **toBeDestroted** 指向 **oldThread**
4.	若 **oldThread** 為user program則old thread 的state和檢查stack overflow
5.	switch to **nextThread** 並將該thread的status設為 **RUNNING** , and call the context switch routine, **SWITCH()**
6.	when **SWITCH()** return, disable interrupt並檢查是否有thread需要被delete掉(Terminate)
7.	恢復 **oldThread** 的相關status資料(userRegisters,pagetable….)

# 1-3. Running→Waiting (Note: only need to consider console output as an example)

### SynchConsoleOutput::PutChar(char)
```javascript=
void SynchConsoleOutput::PutChar(char ch) {
  lock->Acquire();
  consoleOutput->PutChar(ch);
  waitFor->P();
  lock->Release();
}
```
**※ PutChar()本身包含了ConsoleOutput->PutChar(), 所以ConsoleOutPut的每成功PutChar之後就會呼叫SynchConsoleOutput來進行同步**
1.	先做 **lock->Acquire()** , 讓currentThread持有目前輸出權
2.	呼叫 **ConsoleOutput->PutChar()** , 當 **WriteFile** 結束之後會將 **ConsoleOutput** 加入interrupt pending list當console time到了之後就會執行 **ConsoleOutPut->Callback()**

    i.	**ConsoleOutPut->CallBack()** 中會呼叫 **callWhenDone->CallBack()** , 這就是 **SynchConsoleOutput::CallBack()**
    
    ii.	**SynchConsoleOutput::CallBack()** 做了 **waitFor->V()**, semaphore++
    
3.	接下來執行 **waitFor->P()** , semaphore- -
4.	呼叫 **lock->Release()** ,釋放lock

### Semaphore::P()
```javascript=
void
Semaphore::P()
{
	DEBUG(dbgTraCode, "In Semaphore::P(), " << kernel->stats->totalTicks);
    Interrupt *interrupt = kernel->interrupt;
    Thread *currentThread = kernel->currentThread;
    
    // disable interrupts
    IntStatus oldLevel = interrupt->SetLevel(IntOff);	
    
    while (value == 0) { 		// semaphore not available
	    queue->Append(currentThread);	// so go to sleep
	    currentThread->Sleep(FALSE);
    } 
    value--; 			// semaphore available, consume its value
   
    // re-enable interrupts
    (void) interrupt->SetLevel(oldLevel);	
}
```
1.	Disable interrupt,儲存 **oldLevel** 
2.	若value == 0, 代表目前都正在被使用, 需要等待, 所以將目前的thread加進queue裡面並讓thread go to sleep
3.	當semaphore > 0時, 跳出while loop並value - -
4.	將interrupt level恢復為 **oldLevel** 

### SynchList<T\>::Append(T)
```javascript=
void
SynchList<T>::Append(T item)
{
    lock->Acquire();		// enforce mutual exclusive access to the list 
    list->Append(item);
    listEmpty->Signal(lock);	// wake up a waiter, if any
    lock->Release();
}
```
1.	先做 **lock->Acquire()** , 讓currentThread持有access list的權利
2.	將item加入list中
3.	呼叫 **listEmpty->Signal()** 去wake up正在等待此condition的thread(如果有的話)
4.	呼叫 **lock->Release()** ,釋放lock

### Thread::Sleep(bool)
```javascript=
void
Thread::Sleep (bool finishing)
{
    Thread *nextThread;
    
    ASSERT(this == kernel->currentThread);
    ASSERT(kernel->interrupt->getLevel() == IntOff);
    
    DEBUG(dbgThread, "Sleeping thread: " << name);
    DEBUG(dbgTraCode, "In Thread::Sleep, Sleeping thread: " << name << ", " << kernel->stats->totalTicks);

    status = BLOCKED;
	//cout << "debug Thread::Sleep " << name << "wait for Idle\n";
    while ((nextThread = kernel->scheduler->FindNextToRun()) == NULL) {
		kernel->interrupt->Idle();	// no one to run, wait for an interrupt
	}    
    // returns when it's time for us to run
    kernel->scheduler->Run(nextThread, finishing); 
}
```
1.	檢查要sleep的thread是否為current thread
2.	Disable interrupt
3.	將current thread的state設為 **BLOCKED** 
4.	透過呼叫 **FindNextToRun()** 去找到ready list中的thread(用 ***nextThread** 存取位址)並透過 **Run()** 把current thread和next thread做context switch
5.	若ready list為空, 則呼叫 **kernel->interrupt->Idle()** 等待pending interrupt或直接終止程式

### Scheduler::FindNextToRun()
_※和1-2的 **FindNextToRun** 一樣_
1.	檢查interrupt是否disabled
2.	Return 一個ready list的thread,若list為空則return NULL, 否則remove ready list的最前面一個thread並將它return

### Scheduler::Run(Thread*, bool)
_※這裡的 **Run()** 和1-2的一樣，但這邊的 **finishing** 變數是false, 因為old thread並沒有結束，只是因為資源目前正在被使用所以暫時sleep(**P()**呼叫**Sleep(FALSE)**)_
1.	用 ***oldThread** 儲存current thread的位址
2.	check whether interrupt is disabled
3.	if the old thread is finished, 將 **toBeDestroted** 指向 **oldThread** 
4.	若 **oldThread** 為user program則恢復old thread 的state和檢查stack overflow
5.	switch to **nextThread** 並將該thread的status設為 **RUNNING** , and call the context switch routine, **SWITCH()**
6.	when **SWITCH()** return, disable interrupt並檢查是否有thread需要被delete掉(Terminate)
7.	恢復 **oldThread** 的相關status資料(userRegisters,pagetable….)

# 1-4. Waiting→Ready (Note: only need to consider console output as an example)

### Semaphore::V()
```javascript=
void
Semaphore::V()
{
	DEBUG(dbgTraCode, "In Semaphore::V(), " << kernel->stats->totalTicks);
    Interrupt *interrupt = kernel->interrupt;
    
    // disable interrupts
    IntStatus oldLevel = interrupt->SetLevel(IntOff);	
    
    if (!queue->IsEmpty()) {  // make thread ready.
	    kernel->scheduler->ReadyToRun(queue->RemoveFront());
    }
    value++;
    
    // re-enable interrupts
    (void) interrupt->SetLevel(oldLevel);
}
```
1.	Disable interrupt, 儲存目前的interrupt level
2.	將BLOCK的thread加進ready list並將status轉為 **READY**
3.	Value + +
4.	將interrupt level設為 **oldLevel** 

### Scheduler::ReadyToRun(Thread*)
_※和1-1的 **ReadyToRun** 一樣_
1.	檢查interrupt是否disabled
2.	將傳入的thread的status設為 **READY**
3.	將該thread append進ready list

# 1-5. Running→Terminated (Note: start from the Exit system call is called)

### ExceptionHandler(ExceptionType) case SC_Exit
1.	從 **OneInstructuion()** 中呼叫 **RaiseException(SyscallException, 0)** 
2.	**RaiseException()** 呼叫 **ExceptionHandler()** , 最後來到 **SC_Exit** 的case
3.	在這個case中, 會呼叫 **kernel->currentThread->Finish()** 來結束目前的thread

### Thread::Finish()
```javascript=
void
Thread::Finish ()
{
    (void) kernel->interrupt->SetLevel(IntOff);		
    ASSERT(this == kernel->currentThread);
    
    DEBUG(dbgThread, "Finishing thread: " << name);
    Sleep(TRUE);				// invokes SWITCH
    // not reached
}
```
1.	Disable interrupt
2.	檢查呼叫此函數的thread是否為current thread
3.	呼叫 **Sleep(TRUE)** , 傳入true代表做完context switch之後會由其他的thread來delete掉目前正在執行的thread

### Thread::Sleep(bool)
_※這裡的 **Sleep()** 和1-3一樣, 但是傳入的bool值為false_
1.	檢查要sleep的thread是否為current thread
2.	Disable interrupt
3.	將current thread的state設為 **BLOCKED** 
4.	透過呼叫 **FindNextToRun()** 去找到ready list中的thread(用 ***nextThread存取位址** )並透過 **Run()** 把current thread和next thread做context switch
5.	若ready list為空, 則呼叫 **kernel->interrupt->Idle()** 等待pending interrupt或直接終止程式

### Scheduler::FindNextToRun()
_※和1-2的 **FindNextToRun** 一樣_
1.	檢查interrupt是否disabled
2.	Return 一個ready list的thread,
若list為空則return NULL, 否則remove ready list的最前面一個thread並將它return

### Scheduler::Run(Thread*, bool)
_※這裡的 **Run()** 和1-2的一樣, 但這邊的 **finishing** 變數是true, 因為 **oldThread** 已經結束執行, 而當 **SWITCH()** return時, 會進入 **CheckToBeDestroyed()** , 這時候 **oldThread** 就會被摧毀掉, 所以之後的if判斷式就不會成立了_
1.	用 ***oldThread** 儲存current thread的位址
2.	check whether interrupt is disabled
3.	if the old thread is finished, 將 **toBeDestroted** 指向 **oldThread** 
4.	若 **oldThread** 為user program則恢復old thread 的state和檢查stack overflow
5.	switch to **nextThread** 並將該thread的status設為 **RUNNING** , and call the context switch routine, **SWITCH()**
6.	when **SWITCH()** return, disable interrupt並檢查是否有thread需要被delete掉(Terminate)
7.	恢復 **oldThread** 的相關status資料(userRegisters,pagetable….)

# 1-6. Ready→Running

### Scheduler::FindNextToRun()
_※和1-2的 **FindNextToRun** 一樣_
1.	檢查interrupt是否disabled
2.	Return 一個ready list的thread, 若list為空則return NULL, 否則remove ready list的最前面一個thread並將它return

### Scheduler::Run(Thread*, bool)
_※這裡的 **Run()** 和1-2的一樣_
1.	用 ***oldThread** 儲存current thread的位址
2.	check whether interrupt is disabled
3.	if the old thread is finished, 將 **toBeDestroted** 指向 **oldThread** 
4.	若 **oldThread** 為user program則恢復old thread 的state和檢查stack overflow
5.	switch to **nextThread** 並將該thread的status設為RUNNING, and call the context switch routine, **SWITCH()** 
6.	when **SWITCH()** return, disable interrupt並檢查是否有thread需要被delete掉(Terminate)


### SWITCH(Thread*, Thread*)
※ **SWITCH()** 的實作主要透過 **switch.h** 的definition和 **thread.h** 的extern宣告來輔助
1. **switch.h**
    ```c=
    #ifdef x86
    #define _ESP     0
    #define _EAX     4
    #define _EBX     8
    #define _ECX     12
    #define _EDX     16
    #define _EBP     20
    #define _ESI     24
    #define _EDI     28
    #define _PC      32
    #define PCState         (_PC/4-1)
    #define FPState         (_EBP/4-1)
    #define InitialPCState  (_ESI/4-1)
    #define InitialArgState (_EDX/4-1)
    #define WhenDonePCState (_EDI/4-1)
    #define StartupPCState  (_ECX/4-1)
    #define InitialPC       %esi
    #define InitialArg      %edx
    #define WhenDonePC      %edi
    #define StartupPC       %ecx
    #endif
    ```
    * 在 **switch.h** 內宣告register的位置
2. **thread.h** 外部宣告(extern)
    ```c=
    extern "C" {
        void ThreadRoot();
        void SWITCH(Thread *oldThread, Thread *newThread);
    }
    ```
    * 透過extern的宣告以及compiler的輔助，使得x86組語能夠與C語言互相呼叫
3. **switch.s** 實作細節
    ```assembly=
    #include "switch.h"
    #ifdef x86
            .text
            .align  2
            .globl  ThreadRoot
            .globl  _ThreadRoot	
    _ThreadRoot:	
    ThreadRoot:
            pushl   %ebp
            movl    %esp,%ebp
            pushl   InitialArg
            call    *StartupPC
            call    *InitialPC
            call    *WhenDonePC
            # NOT REACHED
            movl    %ebp,%esp
            popl    %ebp
            ret
            .comm   _eax_save,4
            .globl  SWITCH
            .globl  _SWITCH
    _SWITCH:		
    SWITCH:
            movl    %eax,_eax_save          
            movl    4(%esp),%eax            
            movl    %ebx,_EBX(%eax)         
            movl    %ecx,_ECX(%eax)
            movl    %edx,_EDX(%eax)
            movl    %esi,_ESI(%eax)
            movl    %edi,_EDI(%eax)
            movl    %ebp,_EBP(%eax)
            movl    %esp,_ESP(%eax)         
            movl    _eax_save,%ebx          
            movl    %ebx,_EAX(%eax)         
            movl    0(%esp),%ebx            
            movl    %ebx,_PC(%eax)          
            movl    8(%esp),%eax            
            movl    _EAX(%eax),%ebx         
            movl    %ebx,_eax_save          
            movl    _EBX(%eax),%ebx         
            movl    _ECX(%eax),%ecx
            movl    _EDX(%eax),%edx
            movl    _ESI(%eax),%esi
            movl    _EDI(%eax),%edi
            movl    _EBP(%eax),%ebp
            movl    _ESP(%eax),%esp         
            movl    _PC(%eax),%eax          
            movl    %eax,0(%esp)            
            movl    _eax_save,%eax
            ret
    #endif // x86
    ```
### **以x86的架構來討論:** 
*※在 **StackAllocate()** 中，我們已經將未來要執行的函式的address放進Host CPU所對應的registers裡面了
大致上來說switch所做的事就是將 **oldThread** 的目前資料儲存到指向的內存中並且記住return address, 然後再把 **newThread** 所指向的內存中的值load到Host CPU的register中, 然後會從 **newThread** 之前的return address開始執行 **newThread***

* 4(%esp)為oldThread的原本位置
* 8(%esp)為newThread的原本位置
* 0(%esp)為return address
* ecx : 指向startup function (interrupt enable) 
* edx : 包含initial argument to thread function
* esi : 指向thread function
* edi : 指向 **Thread::Finish()** 

1. 一開始 **movl  %eax,_eax_save** 先儲存 **eax** , 再來找到 **oldThread** 的pointer儲存在 **eax** 中
2. 透過 **eax** 找到 **oldThread** 的內存的位置, 並將該thread的目前資訊儲存到它自己的內存空間中
3. 再來儲存 **eax** 和它的return address
4. 接下來找 **newThread**, 和前面的方法一樣, 透過儲存了 **newThread** 的pointer的 **eax** 來的restore **newThread** 的資料到Host CPU的register中
5. 再將入口地址 **eax** 放到0(%esp), 之後即開始從 **newThread** 的return address執行 **newThread** 

### (depends on the previous process state)
**SWITCH()** return之後, 有幾種case:
1.	**oldThread** 原本為 **BLOCKED** , **finising** 參數為true => **oldThread** 被delete掉, **Run()** return
2.	**oldThread** 原本為 **BLOCKED** , **finising** 參數為false => restore **oldThread** , 檢查當初造成被BLOCKED的事件是否已經結束(**waitFor->P()** 等等)
3.	**oldThread** 原本為 **RUNNING** , 有可能因為time interrupt使該thread暫時先將CPU yield給其他thread, state變成 **READY** , 當context switch完之後返回至 **Machine::Run()** 

### for loop in Machine::Run()
1.	同1-2的敘述 **Machine::Run()** 就是在模仿MIPS架構的CPU每一條指令的每一個tick的執行
2.	thread可能因為在 **OneTick()** 中的 **CheckIfDue()** 中發現interrupt, 所以造成 **Yield()** 被呼叫
3.	thread可能因為 **semaphore value == 0** 而去sleep, 讓出CPU
4.	thread執行完畢, 透過 **SC_Exit** 讓出CPU然後被delete

# Difficulties

這次的trace code描述到的相關概念比較複雜，有感於英文能力不足，所以我們選擇了中因即砸的方式來描述每一個code path，而這次的困難點在於semaphore的抽象概念和SWITCH的組合語言的部分，雖然semaphore在概念上比較易於瞭解，但是在code地的實作方面比較複雜，所以trace code的時候比較辛苦，SWITCH的解說則是因為對組語的不熟悉加上網路上的資料偏少，所以花了一些時間才大概了解了實作概念
