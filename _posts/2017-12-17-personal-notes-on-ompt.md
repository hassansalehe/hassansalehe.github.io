---
layout: post
title: "Personal Notes on OpenMP's OMPT"
date: 17-12-2017
comments: true
---
I have been implementing a correctness tool for OpenMP tasks and I have 
been keen to look at the  [OMPT specification
](https://github.com/OpenMPToolsInterface/OMPT-Technical-Report/blob/master/ompt-tr.pdf). 
Below is a short summary of features from the specification which are relevant 
to OpenMP tasks.

Note: We can actually treat every 
[OpenMP implicit](https://docs.oracle.com/cd/E19205-01/820-7883/6nj43o69j/index.html) 
task as a task.

## 1. There is a callback for a task create 
 Envoked when a task or target construct is encountered
 Executed in the contect of the creator task

```cpp
  ompt_event_task_create(...)

  typedef void (*ompt_task_create_callback_t) (
      ompt_data_t *parent_task_data,    /* parent task data (tool controlled) */
      const ompt_frame_t *parent_frame, /* frame data for parent task */
      ompt_data_t *new_task_data,       /* created task’s data (tool controlled) */
      ompt_task_type_t type,            /* type of task being created */
      _Bool has_dependences,            /* task has data dependences */
      const void *codeptr_ra            /* return address of api call */
  );
```

## 2. There is a callback for a task schedule
  The OpenMP runtime invokes this callback after it completes 
  or suspends one task and before it schedules another task. 
  Executes in the context of the newly-scheduled task.

```cpp
  ompt_event_task_schedule(
      prior_task_data,          /* indicates the prior task */
      bool prior_completed,     /* is set if the prior task completed */
      next_task_data            /* indicates the task being scheduled */
  )

  typedef void (*ompt_task_schedule_callback_t) (
      ompt_data_t *prior_task_data, /* descheduled task data (tool controlled) */
      _Bool prior_completed,        /* true if prior task completed*/
      ompt_data_t *next_task_data   /* scheduled task data (tool controlled) */
  );
```

## 3. There is a callback for an implicit task begin and end
  The OpenMP runtime invokes this callback with endpoint=ompt_scope_begin
  after an implicit task is fully initialized but before the task begins 
  to work. 
  
  The OpenMP runtime invokes this callback with the endpoint=ompt_scope_end 
  after the implicit task executes its closing synchronization barrier but
  before the task is destroyed. This callback executes in the context of 
  the implicit task.
  
```cpp
  ompt_event_implicit_task(...)
  
  typedef void (*ompt_scoped_implicit_callback_t) (
      ompt_scope_endpoint_t endpoint,  /* begin or end */
      ompt_data_t *parallel_data,      /* parallel data (tool controlled) */
      ompt_data_t *task_data,          /* task data (tool controlled) */
      uint32_t thread_num              /* OMP thread num */
  );
```

## 4. There is a callback for task dependency
   If a task has any dependences with respect to data objects that constrain 
   its ordering with respect to other tasks, the OpenMP runtime invokes this 
   callback immediately after the callback announcing the task’s creation to 
   announce its dependences with respect to data objects.
   
```cpp
  ompt_event_task_dependences(...)
 
  typedef void (*ompt_task_dependences_callback_t) (
      ompt_data_t *task_data,             /* task data (tool controlled) */
      const ompt_task_dependence_t *deps, /* vector of task dependences */
      int ndeps                           /* number of dependences */
  );
```

## 5. There is callback for task dependence pair

The OpenMP runtime invokes this callback to report a dependence between 
a producer (src_task_data) and a consumer (sink_task_data) that blocked 
execution of the consumer. This callback will occur before the consumer 
knows that the dependence is satisfied. This may happen early or late. 

Note: this callback is used only to report blocking dependences between 
sibling tasks whose lifetimes overlap. No callback will occur if a 
producer task finishes before a consumer task is created.

```cpp
  ompt_event_task_dependence_pair(...)
  
  typedef void (*ompt_task_dependence_callback_t) (
      ompt_data_t *src_task_data,   /* dependence source task data (tool controlled) */
      ompt_data_t *sink_task_data   /* dependence sink task data (tool controlled) */
  );
```

## 6. Extra notes
* Threads, parallel regions, task regions, target regions, and target 
operations are represented by unique identifiers of type ompt_data_t.

```cpp
  typedef uint64_t ompt_id_t;
  typedef union ompt_data_u {
      ompt_id_t id;             /* integer ID under tool control */
      void *ptr;                /* pointer under tool control */
  } ompt_data_t;
  
  /* initial value of ompt_data_t instances provided by the runtime */
  ompt_data_t ompt_data_none = {.id=0}; 
```

This type allows tools to either attach tool-specific data to the 
aforementioned constructs or to maintain a tool-specifc integer ID. 
Both options require the runtime to pass this identifiers by reference 
to the corresponding callbacks. The initial value of an identifier 
is ompt_data_none. This allows tools to detect if the identifier has 
already been initialzed. Tools may assign the identifier a different value.
It is the tool’s responsibility to maintain the resources it assigns to 
the identifiers. If the runtime needs to report an invalid identifier, 
it passes a NULL pointer to the callback or returns a NULL pointer from 
an inquiry API function.

* Thread Identifier:

Each OpenMP thread has an associated identifier of type ompt_data_t. On 
thread creation, the runtime library initializes the thread identifier 
to ompt_data_none. Thread related event callbacks provide the identifier 
by reference in order to let a tool change its value. A thread identifier 
can be retrieved on demand by invoking the ompt_get_thread_data function. 
To indicate an invalid identifier, this function returns a NULL pointer.
```cpp
   OMPT_API ompt_data_t *ompt_get_thread_data(void);
```

* Task Region Identifier:

Each OpenMP task has an associated identifier of type ompt_data_t. Task 
identifiers are assigned to initial, implicit, explicit, and target tasks. 
On task region creation, the runtime library initializes the task region
identifier to ompt_data_none. Task region related event callbacks provide 
the identifier by reference in order to let a tool change its value. A 
task region identifier can be retrieved on demand by invoking the 
ompt_get_task_info function (described in Section 5.5). To indicate an 
invalid identifier, this function returns a NULL pointer.

Function ompt_get_task_info provides information about the task, if any, 
at the specified ancestor level in the current execution context.

```cpp
  OMPT_API _Bool ompt_get_task_info(
      int ancestor_level,             /* 0 refers to the current task*/
      ompt_task_type_t *type,
      ompt_data_t **task_data,
      ompt_frame_t **task_frame,
      ompt_data_t **parallel_data,
      uint32_t *thread_num
  );
```

Ancestor level 0 refers to the current task; information about ancestor
tasks in the current execution context may be queried at higher ancestor 
levels. Function ompt_get_task_info returns a Boolean, which indicates
whether there is a task at the specified ancestor level. This function 
is async signal safe.

A task may be of type initial, implicit, explicit, target, or degenerate. 
If the task at the specified level is degenerate, the address returned in 
*task_data will be NULL. A degenerate task will be associated with the 
enclosing parallel region. If the thread invoking this function is outside 
any parallel region, the address returned in *parallel_data will be NULL. 

A tool can check if the task region or parallel region identifier are in 
their initial states by comparing against ompt_data_none. A tool may modify 
the identifiers. The value returned in thread_num indicates the number of 
the OpenMP thread executing the task in the parallel region to which the 
task belongs. 

Using values inside ompt_frame_t objects returned by calls to 
ompt_get_task_info, a tool can analyze frames in the call stack and identify 
ones that exist on behalf of the runtime system. 3 This capability enables a 
tool to map from an implementation-level view of the call stack back to a 
source-level view that is easier for application developers to understand. 
Appendix B discusses an example that illustrates the use of ompt_frame_t 
objects with multiple threads and nested parallelism.

* Task types:
```cpp
  typedef enum ompt_task_type_e {
      ompt_task_initial     = 1, 
      ompt_task_implicit    = 2,
      ompt_task_explicit    = 3,
      ompt_task_target      = 4,
      ompt_task_degenerate  = 5
  } ompt_task_type_t;
```

* Task dependence types:
```cpp
  typedef enum ompt_task_dependence_flag_e {
      // a two bit field for the dependence type
      ompt_task_dependence_type_out     = 1,
      ompt_task_dependence_type_in      = 2,
      ompt_task_dependence_type_inout   = 3
  } ompt_task_dependence_flag_t;
```

* Task dependence address:
```cpp
  typedef struct ompt_task_dependence_s {
      void *variable_addr;
      uint32_t dependence_flags;
  } ompt_task_dependence_t;
```

* Task regions callbacks:
```cpp
  typedef void (*ompt_task_create_callback_t) (
      ompt_data_t *parent_task_data,    /* parent task data (tool controlled) */
      const ompt_frame_t *parent_frame, /* frame data for parent task */
      ompt_data_t *new_task_data,       /* created task’s data (tool controlled) */
      ompt_task_type_t type,            /* type of task being created */
      _Bool has_dependences,            /* task has data dependences */
      const void *codeptr_ra            /* return address of api call */
  );

  typedef void (*ompt_task_dependences_callback_t) (
      ompt_data_t *task_data,             /* task data (tool controlled) */
      const ompt_task_dependence_t *deps, /* vector of task dependences */
      int ndeps                           /* number of dependences */
  );

  typedef void (*ompt_task_dependence_callback_t) (
      ompt_data_t *src_task_data,       /* dependence source task data (tool controlled) */
      ompt_data_t *sink_task_data       /* dependence sink task data (tool controlled) */
  );
```
