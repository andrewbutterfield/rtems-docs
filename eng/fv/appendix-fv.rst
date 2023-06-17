.. SPDX-License-Identifier: CC-BY-SA-4.0

.. Copyright (C) 2022 Trinity College Dublin

Appendix: RTEMS Formal Model Guide
**********************************

This appendix covers the various formal models of RTEMS that are currently in
existence. It serves two purposes:
one is to provide detailed documentation of each model,
while the other is provide a guide into how to go about developing and deploying such models.

The general approach followed here is to start by looking at the API documentation and identifying the key data-structures and function prototypes.
These are then modelled appropriately in Promela.
Then, general behavior patterns of interest are identified, 
and the Promela model is extended to provide those patterns.
A key aspect here is exploiting the fact that Promela allows non-deterministic choices to be specified, which gives the effect of producing arbitrary orderings of model behavior.
All of this leads to a situation were the SPIN model-checker can effectively generate scenarios for all possible interleavings.
The final stage is mapping those scenarios to RTEMS C test code,
which has two parts: generating machine-readable output from SPIN, and developing the refinement mapping from that output to C test code.

Some familiarity is assumed here with the Software Test Framework section in this document.

The following models are included in the directory ``formal/promela/models/`` 
at the top-level in ``rtems-central``:

Chains API (``chains/``)
    Models the unprotected chain append and get API calls in the Classic
    Chains API Guide. This was an early model to develop the basic methodology.

Events Manager (``events/``)
    Models the behaviour of all the API calls in the Classic Events Manager API
    Guide. This had to tackle real concurrency and deal with multiple CPUs and priority
    issues.

Barrier Manager (``barriers/``)
    Models the behaviour of all the API calls in then Classic Barrier Manager API.

Message Manager (``messages/``)
    Models the create, send and receive API calls in the Classic Message Manager API.

At the end of this guide is a section that discusses various issues that should be tackled in future work.


*Gedare:* **This all reads like a project report, rather than a manual. 
Although the references to the completed MSc dissertations are appreciated and 
useful, they should be better integrated where the work that was done is 
described within the manual itself. The start of this appendix should better 
explain what is being shown here, i.e., 
the detailed examples of the above modelings. 
You might also link to the relevant sections.**


Testing Chains
--------------

Documentation:  Chains section in the RTEMS Classic API Guide.

Model Directory: ``formal/promela/models/chains``.

Model Name: ``chains-api-model``.

The Chains API provides a doubly-linked list data-structure, optimised for fast
operations in an SMP setting. It was used as proof of concept exercise,
and focussed on just two API calls: ``rtems-chain-append-unprotected``
and ``rtems-chain-get-unprotected`` (hereinafter just ``append`` and ``get``).


API Model
^^^^^^^^^

File: ``chains-api-model.pml``

While smart code optimization techniques are very important for RTEMS code,
the focus when constructing formal models is on functional correctness,
not performance. What is required is the simplest, most obviously correct model.

The ``append`` operation adds new nodes on the end of the list,
while ``get`` removes and returns the node at the start of the list.
The Chains API has many other operations that can add/remove nodes at either end, or somewhere in the middle, but these are considered out of scope.

Data Structures
~~~~~~~~~~~~~~~

There are no pointers in Promela, so we have to use arrays, 
with array indices modelling pointers.
With just ``append`` and ``get``, an array can be used to implement a collection
of nodes in memory.
A ``Node`` type is defined that has next and previous indices, 
plus an item payload.
Access to the node list is via a special control node with head and tail pointers.
In the model, an explicit size value is added to this control node,
to allow the writing of properties about chain length,
and to prevent array out-of-bound errors in the model itself.
We assume a single ``chain``, 
with list node storage statically allocated in ``memory``.

.. code:: c

  #define PTR_SIZE 3
  #define MEM_SIZE 8

  typedef Node {
    unsigned nxt  : PTR_SIZE
  ; unsigned prv  : PTR_SIZE
  ; byte     itm
  }
  Node memory[MEM_SIZE] ;
  
  typedef Control {
    unsigned head : PTR_SIZE; 
    unsigned tail : PTR_SIZE; 
    unsigned size : PTR_SIZE
  }
  Control chain ;

While there are 8 memory elements, element 0 is inaccessible, 
as the index 0 is treated like a ``NULL`` pointer.

Function Calls
~~~~~~~~~~~~~~

The RTEMS prototype for ``append`` is:

.. code:: c 

  void rtems_chain_append_unprotected(
      rtems_chain_control *the_chain,
      rtems_chain_node    *the_node
  );

Its implementation starts by checking that the node to be appended is "off
chain", before performing the append.
The model is designed to satisfy this property so the check is not modelled.
Also, the Chains documentation is not clear about certain error cases.
As this is a proof of concept exercise, these details are not modelled.

A Promela inline definition ``append`` models the desired behavior,
simulating C pointers with array addresses. Here ``ch`` is the chain argument,
while ``np`` is a node index.
The model starts by checking that the node pointer is not ``NULL``,
and that there is room in ``memory`` for another node.
These are to ensure that the model does not have any runtime errors.
Doing a standard model-check of this model finds no errors,
which indicates that those assertions are never false.

.. code:: c

  inline append(ch,np) {
    assert(np!=0); assert(ch.size < (MEM_SIZE-1));
    if
    :: (ch.head == 0) -> ch.head = np; ch.tail = np; ch.size = 1;
                         memory[np].nxt = 0; memory[np].prv = 0;
    :: (ch.head != 0) -> memory[ch.tail].nxt = np; memory[np].prv = ch.tail;
                         ch.tail = np; ch.size = ch.size + 1;
    fi
  }

The RTEMS prototype for ``get`` is:

.. code:: c 

  rtems_chain_node *rtems_chain_get_unprotected(
    rtems_chain_control *the_chain
  );

It returns a pointer to the node, with ``NULL`` returned if the chain is empty.

Promela inlines involve textual substitution, 
so the concept of returning a value makes no sense.
For ``get``,  the model is that of a statement that assigns the return value to
a variable. Both the function argument and return variable name are passed as parameters:

.. code:: c 

  /* np = get(ch); */
  inline get(ch,np) {
    np = ch.head ;
    if
      :: (np != 0) ->
          ch.head = memory[np].nxt;
          ch.size = ch.size - 1;
          // memory[np].nxt = 0
      :: (np == 0) -> skip
    fi
    if
      :: (ch.head == 0) -> ch.tail = 0
      :: (ch.head != 0) -> skip
    fi
  }

Behavior patterns
^^^^^^^^^^^^^^^^^

File: ``chains-api-model.pml``

A key feature of using a modelling language like Promela is that it has both
explicit and implicit non-determinism. This can be exploited so that SPIN will
find all possible interleavings of behavior.

The Chains API model consists of six processes, three which perform ``append``,
and three that perform ``get``, waiting if the chain is empty. This model relies
on implicit non-determinism, in that the SPIN scheduler can choose and switch 
between any unblocked process at any point. There is no explicit non-determinism
in this model.

Promela process ``doAppend`` takes node index ``addr`` and a value ``val`` as
parameters. It puts ``val`` into the node indexed by ``addr``,
then calls ``append``, and terminates. 
It is all made atomic to avoid unnecessary internal interleaving of operations because unprotected versions of API calls should only be used when interrupts
are disabled.

.. code:: c

  proctype doAppend(int addr; int val) {
    atomic{ memory[addr].itm = val; 
            append(chain,addr); } ;
  }

The ``doNonNullGet`` process waits for the chain to be non-empty before attempting to ``get`` an element. The first statement inside the atomic
construct is an expression, as a statements, that blocks while it evaluates to
zero. That only happens if ``head`` is in fact zero. The model also has an 
assertion that checks that a non-null node is returned.

.. code:: c

  proctype doNonNullGet() {
    atomic{
      chain.head != 0;
      get(chain,nptr);
      assert(nptr != 0);
    } ;
  }

All processes terminate after they have performed their (sole) action.

The top-level of a Promela model is an initial process declared by the``init`` construct. This initializes the chain as empty and then runs all six processes concurrently. It then uses the special ``_nr_pr`` variable to wait for all six
processes to terminate. A final assertion checks that the chain is empty.

.. code:: c

  init {
    pid nr;
    chain.head = 0; chain.tail = 0; chain.size = 0 ;
    nr = _nr_pr;  // assignment, sets `nr` to current number of procs
    run doAppend(6,21);
    run doAppend(3,22);
    run doAppend(4,23);
    run doNonNullGet();
    run doNonNullGet();
    run doNonNullGet();
    nr == _nr_pr; // expression, waits until number of procs equals `nr`
    assert (chain.size == 0);
  }

Simulation of this model will show some execution sequence in which the appends
happen in a random order, and the gets also occur in a random order, whenever
the chain is not empty. All assertions are always satisfied, including the last
one above. Model checking this model explores all possible interleavings and reports no errors of any kind. In particular, when the model reaches the last
assert statement, the chain size is always zero.

SPIN uses the C pre-processor, and generates the model as a C program. This 
model has a simple flow of control: basically execute each process once in an
almost arbitrary order, assert that the chain is empty, and terminate. Test
generation here just requires the negation of the final assertion to get all
possible interleavings. The special C pre-processor definition ``TEST_GEN`` is
used to switch between the two uses of the model. The last line above is
replaced by:

.. code:: c

  #ifdef TEST_GEN
    assert (chain.size != 0);
  #else
    assert (chain.size == 0);
  #endif

A test generation run can then be invoked by passing in ``-DTEST_GEN`` as a 
command-line argument.

Annotations
^^^^^^^^^^^

File: ``chains-api-model.pml``

The model needs to have ``printf`` statements added to generation the
annotations used to perform the test generation.

This model wraps each of six API calls in its own process, so that model
checking can generate all feasible interleavings. However, the plan for the test code is that it will be just one RTEMS Task, that executes all the API
calls in the order determined by the scenario under consideration. All the 
annotations in this model specify ``0`` as the Promela process identifier.

Data Structures
~~~~~~~~~~~~~~~

Annotations have to be provided for any variable or datastructure declarations
that will need to have corresponding code in the test program.
These have to be printed out as the model starts to run.
For this model, the ``MAX_SIZE`` parameter is important,
as are the variables ``memory``, ``nptr``, and ``chain``:

.. code:: c

  printf("@@@ 0 NAME Chain_AutoGen\n")
  printf("@@@ 0 DEF MAX_SIZE 8\n");
  printf("@@@ 0 DCLARRAY Node memory MAX_SIZE\n");
  printf("@@@ 0 DECL unsigned nptr NULL\n")
  printf("@@@ 0 DECL Control chain\n");

At this point, a parameter-free initialization annotation is issued. This should
be refined to C code that initializes the above variables.

.. code:: c

  printf("@@@INIT\n");

Function Calls
~~~~~~~~~~~~~~

For ``append``, two forms of annotation are produced. One uses the ``CALL``
format to report the function being called along with its arguments. The other
form reports the resulting contents of the chain.

.. code:: c

   proctype doAppend(int addr; int val) {
     atomic{ memory[addr].itm = val; append(chain,addr);
             printf("@@@ 0 CALL append %d %d\n",val,addr); 
             show_chain(); 
           } ;
   }

The statement ``show_chain()`` is an inline function that prints the
contents of the chain after append returns.
The resulting output is multi-line,
starting with ``@@@ 0 SEQ chain``,
ending with ``@@@ 0 END chain``,
and with entries in between of the form ``@@@ 0 SCALAR _ val``
displaying chain elements, line by line.

Something similar is done for ``get``, with the addition of a third annotation
``show_node()`` that shows the node that was got:

.. code:: c

  proctype doNonNullGet() {
    atomic{
      chain.head != 0;
      get(chain,nptr);
      printf("@@@ 0 CALL getNonNull %d\n",nptr);
      show_chain();
      assert(nptr != 0);
      show_node();
    } ;
  }

The statement ``show_node()`` is defined as follows:

.. code:: c

  inline show_node (){
    atomic{
      printf("@@@ 0 PTR nptr %d\n",nptr);
      if
      :: nptr -> printf("@@@ 0 STRUCT nptr\n");
                 printf("@@@ 0 SCALAR itm %d\n", memory[nptr].itm);
                 printf("@@@ 0 END nptr\n")
      :: else -> skip
      fi
    }
  }

It prints out the value of ``nptr``, which is an array index. If it is not zero,
it prints out some details of the indexed node structure.

Annotations are also added to the ``init`` process to show the chain and node.

.. code:: c

  chain.head = 0; chain.tail = 0; chain.size = 0;
  show_chain();
  show_node();
 
Refinement
^^^^^^^^^^

File: ``chains-api-model-rfn.yml``


**NEED TO DESCRIBE HOW TO DESIGN REFINEMENT ENTRIES**

The ``spin2test`` script takes these annotations, along with the YAML
refinement file defined for the model, and proceeds to generate testcode. All
of these annotations have the same ``<pid>``, namely 0, so one test segment of
code is produced. We show some examples of how this works below.

Given ``@@@ 0 NAME Chain_AutoGen`` we lookup `NAME` in the refinement file,
and get the following (which ignores the ``<name>`` parameter in this case):

.. code-block:: c

     const char rtems_test_name[] = "Model_Chain_API";

For ``@@@ 0 DEF MAX_SIZE 8`` we directly output

.. code-block:: c

   #define MAX_SIZE 8

For ``@@@ 0 DCLARRAY Node memory MAX_SIZE`` we lookup ``memory_DCL`` and get
``item {0}[{1}];``. We substitute ``memory`` and ``MAX_SIZE`` to get

.. code-block:: c

   item memory[MAX_SIZE];

For ``INIT`` we lookup ``INIT`` to get

.. code-block:: c

   rtems_chain_initialize_empty( &chain );

The first ``SEQ`` ... ``END`` pair is intended to display the initial chain,
which should be empty. The second shows the result of an ``append`` with one
value in the chain. In both cases, the name ``chain`` is recorded, and for
each ``SCALAR _ val``, the value of ``val`` is printed to a string with a
leading space. When ``@@@ 0 END chain`` is encountered we lookup ``chain_SEQ``
to obtain:

.. code-block:: c

     show_chain( &chain, ctx->buffer );
     T_eq_str( ctx->buffer, "{0} 0" );

Function ``show_chain`` is defined in the preamble C file used in test
generation and is designed to display the chain contents in a string that
matches the one generated here by the processing of ``SEQ`` ... ``SCALAR`` ...
``END``. We substitute the accumulated string in for ``{0}``, which will be
either empty, or just " 23". In the latter case we get the following code:

.. code-block:: c

     show_chain( &chain, ctx->buffer );
     T_eq_str( ctx->buffer, "23 0" );


For ``@@@ 0 CALL append 22 3`` we lookup ``append`` to get

.. code-block:: c

     memory[{1}].val = {0};
     rtems_chain_append_unprotected( &chain, (rtems_chain_node*)&memory[{1}] );

We substitute ``22`` and ``3`` in to get

.. code-block:: c

     memory[3].val = 22;
     rtems_chain_append_unprotected( &chain, (rtems_chain_node*)&memory[3] );


The following is the corresponding excerpt from the generated test-segment:

.. code-block:: c

  // @@@ 0 NAME Chain_AutoGen
  // @@@ 0 DEF MAX_SIZE 8
  #define MAX_SIZE 8
  // @@@ 0 DCLARRAY Node memory MAX_SIZE
  static item memory[MAX_SIZE];
  // @@@ 0 DECL unsigned nptr NULL
  static item * nptr = NULL;
  // @@@ 0 DECL Control chain
  static rtems_chain_control chain;

  //  ===== TEST CODE SEGMENT 0 =====

  static void TestSegment0( Context* ctx ) {
    const char rtems_test_name[] = "Model_Chain_API";

    T_log(T_NORMAL,"@@@ 0 INIT");
    rtems_chain_initialize_empty( &chain );
    T_log(T_NORMAL,"@@@ 0 SEQ chain");
    T_log(T_NORMAL,"@@@ 0 END chain");
    show_chain( &chain, ctx->buffer );
    T_eq_str( ctx->buffer, " 0" );

    T_log(T_NORMAL,"@@@ 0 PTR nptr 0");
    T_eq_ptr( nptr, NULL );
    T_log(T_NORMAL,"@@@ 0 CALL append 22 3");
    memory[3].val = 22;
    rtems_chain_append_unprotected( &chain, (rtems_chain_node*)&memory[3] );

    T_log(T_NORMAL,"@@@ 0 SEQ chain");
    T_log(T_NORMAL,"@@@ 0 SCALAR _ 22");
    T_log(T_NORMAL,"@@@ 0 END chain");
    show_chain( &chain, ctx->buffer );
    T_eq_str( ctx->buffer, " 22 0" );
    ...
  }

Note the extensive use of ``T_log()``, and emitted comments showing the
annotations when producing declarations. These help when debugging models,
refinement files, and the resulting test code. There are plans to provide a
mechanism that can be used to control the level of verbosity involved.


Testing Events
--------------


Documentation:  Event Manager section in the RTEMS Classic API Guide.

Model Directory: ``formal/promela/models/events``.

Model Name: ``event-mgr-model``.

The Event Manager is a central piece of code in RTEMS SMP, being at the basis
of task communication and synchronization. It is used for instance in the
implementation of semaphores or various essential high-level data-structures,
and used in the Scheduling process. At the same time, its implementation is
making use of concurrent features of C11, and contains many unprotected
interactions with the Threads API. Having a Promela model faithfully modelling
the Event Manager code of RTEMS represent thus a real challenge, especially
with respect to formal testing. This application constitutes as well a way to
measure the completeness of our manual and automatic test generation tools
previously developed.

The RTEMS Event Manager was chosen as the second case-study because
it involved concurrency and communication, had a small number of API calls
(just two),
but also had somewhat complex requirements related to task priorities.

The Event Manager allows tasks to send events to,
and receive events from, other tasks.
From the perspective of the Event Manager,
events are just uninterpreted numbers in the range 0..31,
encoded as a 32-bit bitset.

``rtems_event_send(id,event_in)``
  allows a task to send a bitset to a designated task

``rtems_event_receive(event_in,option_set,ticks,event_out)``
  allows a task to specify a desired bitset
  with options on what to do if it is not present.

Most of the requirements are pretty straightforward,
but two were a little more complex,
and drove the more complex parts of the modelling.

1. If a task was blocked waiting to receive events,
   and a lower priority task then sent the events that would wake that
   blocked task,
   then the sending task would be immediately preempted by the receiver task.

2. There was a requirement that explicitly discussed the situation
   where the two tasks involved were running on different processors.


API Model
^^^^^^^^^

File: ``xxx-model.pml``

The RTEMS Event set contains 32 values, but in our model we limit ourselves to
just four, which is enough for test purposes. 

We simplify the ``rtems_option_set`` to just two relevant bits: the timeout
setting (``Wait``, ``NoWait``), and how much of the desired event set will
satisfy the receiver (``All``, ``Any``).

There is no notion of returning values from Promela ``proctype`` or ``inline``
constructs, so we need to have global variables to model return values. Also,
C pointers used to designate where to return a result need to be modelled
by indices into global array variables.

Event Send
~~~~~~~~~~

We start with the notion of when a event receive call is satisfied. The
requirements for both send and receive depend on such satisfaction.

``satisfied(task,out,sat)``
    ``satisfied(task,out,sat)`` checks if a receive has been satisfied. It
    updates its ``sat`` argument to reflect the check outcome.

An RTEMS call ``rc = rtems_event_send(tid,evts)`` is modelled by an inline of
the form:

.. code-block:: c

   event_send(self,tid,evts,rc)

The four arguments are:
 | ``self`` : id of process modelling the task/IDR making call.
 | ``tid``  : id of process modelling the target task of the call.
 | ``evts`` : event set being sent.
 | ``rc``   : updated with the return code when the send completes.

The main complication in the otherwise straightforward model is the requirement
to preempt under certain circumstances.

.. code-block:: c

   inline event_send(self,tid,evts,rc) {
     atomic{
       if
       ::  tid >= BAD_ID -> rc = RC_InvId
       ::  tid < BAD_ID ->
           tasks[tid].pending = tasks[tid].pending | evts
           // at this point, have we woken the target task?
           unsigned got : NO_OF_EVENTS;
           bool sat;
           satisfied(tasks[tid],got,sat);
           if
           ::  sat ->
               tasks[tid].state = Ready;
               printf("@@@ %d STATE %d Ready\n",_pid,tid)
               preemptIfRequired(self,tid) ;
               // tasks[self].state may now be OtherWait !
               waitUntilReady(self);
           ::  else -> skip
           fi
           rc = RC_OK;
       fi
     }
   }


Event Receive
~~~~~~~~~~~~~

An RTEMS call ``rc = rtems_event_receive(evts,opts,interval,out)`` is modelled
by an inline of
the form:

.. code-block:: c

   event_receive(self,evts,wait,wantall,interval,out,rc)

The seven arguments are:
 | ``self`` : id of process modelling the task making call
 | ``evts`` : input event set
 | ``wait`` : true if receive should wait
 | ``what`` : all, or some?
 | ``interval`` : wait interval (0 waits forever)
 | ``out`` : pointer to location for satisfying events when the receive
     completes.
 | ``rc`` : updated with the return code when the receive completes.


There is a small complication, in that we have distinct variables in our model
for receiver options that are combined into a single RTEMS option set. The
actual calling sequence in C test code will be:

.. code-block:: c

   opts = mergeopts(wait,wantall);
   rc = rtems_event_receive(evts,opts,interval,out);

Here ``mergeopts`` is a C function defined in the C Preamble.

.. code-block:: c

   inline event_receive(self,evts,wait,wantall,interval,out,rc){
     atomic{
       printf("@@@ %d LOG pending[%d] = ",_pid,self);
       printevents(tasks[self].pending); nl();
       tasks[self].wanted = evts;
       tasks[self].all = wantall
       if
       ::  out == 0 ->
           printf("@@@ %d LOG Receive NULL out.\n",_pid);
           rc = RC_InvAddr ;
       ::  evts == EVTS_PENDING ->
           printf("@@@ %d LOG Receive Pending.\n",_pid);
           recout[out] = tasks[self].pending;
           rc = RC_OK
       ::  else ->
           bool sat;
           retry:  satisfied(tasks[self],recout[out],sat);
           if
           ::  sat ->
               printf("@@@ %d LOG Receive Satisfied!\n",_pid);
               setminus(tasks[self].pending,tasks[self].pending,recout[out]);
               printf("@@@ %d LOG pending'[%d] = ",_pid,self);
               printevents(tasks[self].pending); nl();
               rc = RC_OK;
           ::  !sat && !wait ->
               printf("@@@ %d LOG Receive Not Satisfied (no wait)\n",_pid);
               rc = RC_Unsat;
           ::  !sat && wait && interval > 0 ->
               printf("@@@ %d LOG Receive Not Satisfied (timeout %d)\n",_pid,interval);
               tasks[self].ticks = interval;
               tasks[self].tout = false;
               tasks[self].state = TimeWait;
               printf("@@@ %d STATE %d TimeWait %d\n",_pid,self,interval)
               waitUntilReady(self);
               if
               ::  tasks[self].tout  ->  rc = RC_Timeout
               ::  else              ->  goto retry
               fi
           ::  else -> // !sat && wait && interval <= 0
               printf("@@@ %d LOG Receive Not Satisfied (wait).\n",_pid);
               tasks[self].state = EventWait;
               printf("@@@ %d STATE %d EventWait\n",_pid,self)
               if
               :: sendTwice && !sentFirst -> Released(sendSema);
               :: else
               fi
               waitUntilReady(self);
               goto retry
           fi
       fi
       printf("@@@ %d LOG pending'[%d] = ",_pid,self);
       printevents(tasks[self].pending); nl();
     }
   }



Behaviour Patterns
^^^^^^^^^^^^^^^^^^

File: ``xxx-model.pml``

The Event Manager model consists of
five Promela processes:

``init``
    The first top-level Promela process that performs initialisation,
    starts the other processes, waits for them to terminate, and finishes.

``System``
    A Promela process that models the behaviour of the operating system,
    in particular that of the scheduler.

``Clock``
    A Promela process used to facilitate modelling timeouts.

``Sender``
    A Promela process used to model the RTEMS sender task.

``Receiver``
    A Promela process used to model the RTEMS receiver task.

We envisage two RTEMS tasks
involved, at most. We use two simple binary semaphores to synchronise the tasks.
We provide some inline definitions to encode (``events``), display
(``printevents``), and subtract (``setminus``) events.

Our Task model only looks at an abstracted version of RTEMS Task states:

``Zombie``
    used to model a task that has just terminated. It can only be deleted.

``Ready``
    same as the RTEMS notion of ``Ready``.

``EventWait``
    is ``Blocked`` inside a call of ``event_receive()`` with no timeout.

``TimeWait``
    is ``Blocked`` inside a call of ``event_receive()`` with a timeout.

``OtherWait``
    is ``Blocked`` for some other reason, which arises in this model when a
    sender gets pre-empted by a higher priority receiver it has just satisfied.


We represent tasks using a datastructure array. As array indices are proxies
here for C pointers, the zeroth array entry is always unused, as we use index
value 0 to model a NULL C pointer.

.. code-block:: c

   typedef Task {
     byte nodeid; // So we can spot remote calls
     byte pmlid; // Promela process id
     mtype state ; // {Ready,EventWait,TickWait,OtherWait}
     bool preemptable ;
     byte prio ; // lower number is higher priority
     int ticks; //
     bool tout; // true if woken by a timeout
     unsigned wanted  : NO_OF_EVENTS ; // EvtSet, those expected by receiver
     unsigned pending : NO_OF_EVENTS ; // EvtSet, those already received
     bool all; // Do we want All?
   };
   Task tasks[TASK_MAX]; // tasks[0] models a NULL dereference

.. code-block:: c

   byte sendrc;            // Sender global variable
   byte recrc;             // Receiver global variable
   byte recout[TASK_MAX] ; // models receive 'out' location.





Task Scheduling
~~~~~~~~~~~~~~~


In order to produce a model that captures real RTEMS Task behaviour, we need
to have mechanisms that mimic the behaviour of the scheduler and other
activities that can modify the execution state of these Tasks. Given a scenario
generated by such a model, we need to add synchronisation to the generated C
code to ensure test has the same execution patterns.

For scheduling we use:

``waitUntilReady``
    ``waitUntilReady(id)`` logs that ``task[id]`` is waiting, and then attempts
    to execute a statement that blocks, until some other process changes
    ``task[id]``\ 's state to ``Ready``. It relies on the fact that if a
    statement blocks inside an atomic block, the block loses its atomic
    behaviour and yields to other Promela processes It is used to model a task
    that has been suspended for any reason.

``preemptIfRequired``
    ``preemptIfRequired(sendid,rcvid)`` is executed, when ``task[rcvid]`` has had its receive request satisfied
    by a send from ``task[sendid]``. It is invoked by the send operation in this
    model. It checks if ``task[sendid]`` should be preempted, and makes it so.
    This is achieved here by setting the task state to ``OtherWait``.

For synchronisation we use simple boolean semaphores, where True means
available, and False means the semaphore has been acquired.

.. code-block:: c

   bool semaphore[SEMA_MAX]; // Semaphore

The synchronisation mechanisms are:


``Obtain(sem_id)``
   call that waits to obtain semaphore ``sem_id``.

``Release(sem_id)``
    call that releases semaphore ``sem_id``

``Released(sem_id)``
    simulates ecosystem behaviour that releases ``sem_id``.

The difference between ``Release`` and ``Released`` is that the first issues
a ``SIGNAL`` annotation, while the second does not.

Scenarios
~~~~~~~~~

We define a number of different scenario schemes that cover various aspects of
Event Manager behaviour. Some schemes involve only one task, and are usually
used to test error-handling or abnormal situations. Other schemes involve two
tasks, with some mixture of event sending and receiving, with varying task
priorities.

For example, an event send operation can involve a target identifier that
is invalid (``BAD_ID``), correctly identifies a receiver task (``RCV_ID``), or
is sending events to itself (``SEND_ID``).

.. code-block:: c

   typedef SendInputs {
     byte target_id ;
     unsigned send_evts : NO_OF_EVENTS ;
   } ;
   SendInputs  send_in[MAX_STEPS];

An event receive operation will be determined by values for desired events,
and the relevant to bits of the option-set parameter.

.. code-block:: c

   typedef ReceiveInputs {
     unsigned receive_evts : NO_OF_EVENTS ;
     bool will_wait;
     bool everything;
     byte wait_length;
   };
   ReceiveInputs receive_in[MAX_STEPS];

We have a range of global variables that define scenarios for both send and
receive. We then have a two-step process for choosing a scenario.
The first step is to select a scenario scheme. The poissible schemes are
defined by the following ``mtype``:

.. code-block:: c

   mtype = {Send,Receive,SndRcv,RcvSnd,SndRcvSnd,SndPre,MultiCore};
   mtype scenario;

One of these is chosen by using a conditional where all alternatives are
executable, so behaving as a non-deterministic choice of one of them.

.. code-block:: c

   if
   ::  scenario = Send;
   ::  scenario = Receive;
   ::  scenario = SndRcv;
   ::  scenario = SndPre;
   ::  scenario = SndRcvSnd;
   ::  scenario = MultiCore;
   fi


Once the value of ``scenario`` is chosen, it is used in another conditional
to select a non-deterministic choice of the finer details of that scenario.

.. code-block:: c

    if
    ::  scenario == Send ->
          doReceive = false;
          sendTarget = BAD_ID;
    ::  scenario == Receive ->
          doSend = false
          if
          :: rcvWait = false
          :: rcvWait = true; rcvInterval = 4
          :: rcvOut = 0;
          fi
          printf( "@@@ %d LOG sub-senario wait:%d interval:%d, out:%d\n",
                  _pid, rcvWait, rcvInterval, rcvOut )
    ::  scenario == SndRcv ->
          if
          ::  sendEvents = 14; // {1,1,1,0}
          ::  sendEvents = 11; // {1,0,1,1}
          fi
          printf( "@@@ %d LOG sub-senario send-receive events:%d\n",
                  _pid, sendEvents )
    ::  scenario == SndPre ->
          sendPrio = 3;
          sendPreempt = true;
          startSema = rcvSema;
          printf( "@@@ %d LOG sub-senario send-preemptable events:%d\n",
                  _pid, sendEvents )
    ::  scenario == SndRcvSnd ->
          sendEvents1 = 2; // {0,0,1,0}
          sendEvents2 = 8; // {1,0,0,0}
          sendEvents = sendEvents1;
          sendTwice = true;
          printf( "@@@ %d LOG sub-senario send-receive-send events:%d\n",
                  _pid, sendEvents )
    ::  scenario == MultiCore ->
          multicore = true;
          sendCore = 1;
          printf( "@@@ %d LOG sub-senario multicore send-receive events:%d\n",
                  _pid, sendEvents )
    ::  else // go with defaults
    fi

We define default values for all the global scenario variables so that the
above code focusses on what differs. The default scenario is a receiver waiting
for a sender of the same priority which sends exactly what was requested.

Sender Process
~~~~~~~~~~~~~~


The sender process then uses the scenario configuration to determine its
behaviour. A key feature is the way it acquires its semaphore before doing a
send, and releases the receiver semaphore when it has just finished sending.
Both these semaphores are initialised in the unavailable state.

.. code-block:: c

   proctype Sender (byte nid, taskid) {

     tasks[taskid].nodeid = nid;
     tasks[taskid].pmlid = _pid;
     tasks[taskid].prio = sendPrio;
     tasks[taskid].preemptable = sendPreempt;
     tasks[taskid].state = Ready;
     printf("@@@ %d TASK Worker\n",_pid);
     if
     :: multicore ->
          // printf("@@@ %d CALL OtherScheduler %d\n", _pid, sendCore);
          printf("@@@ %d CALL SetProcessor %d\n", _pid, sendCore);
     :: else
     fi
     if
     :: sendPrio > rcvPrio -> printf("@@@ %d CALL LowerPriority\n", _pid);
     :: sendPrio == rcvPrio -> printf("@@@ %d CALL EqualPriority\n", _pid);
     :: sendPrio < rcvPrio -> printf("@@@ %d CALL HigherPriority\n", _pid);
     :: else
     fi
   repeat:
     Obtain(sendSema);
     if
     :: doSend ->
       if
       :: !sentFirst -> printf("@@@ %d CALL StartLog\n",_pid);
       :: else
       fi
       printf("@@@ %d CALL event_send %d %d %d sendrc\n",_pid,taskid,sendTarget,sendEvents);
       if
       :: sendPreempt && !sentFirst -> printf("@@@ %d CALL CheckPreemption\n",_pid);
       :: !sendPreempt && !sentFirst -> printf("@@@ %d CALL CheckNoPreemption\n",_pid);
       :: else
       fi
       event_send(taskid,sendTarget,sendEvents,sendrc);
       printf("@@@ %d SCALAR sendrc %d\n",_pid,sendrc);
     :: else
     fi
     Release(rcvSema);
     if
     :: sendTwice && !sentFirst ->
        sentFirst = true;
        sendEvents = sendEvents2;
        goto repeat;
     :: else
     fi
     printf("@@@ %d LOG Sender %d finished\n",_pid,taskid);
     tasks[taskid].state = Zombie;
     printf("@@@ %d STATE %d Zombie\n",_pid,taskid)
   }

Receiver Process
~~~~~~~~~~~~~~~~

The receiver process  uses the scenario configuration to determine its
behaviour. It has the responsibility to trigger the start semaphore to allow
either itself or the sender to start. The start semaphore corresponds to either
the send or receive semaphore, depending on the scenario. The receiver acquires
the receive semaphore before proceeding, and releases the send sempahore when
done.

.. code-block:: c

   proctype Receiver (byte nid, taskid) {

     tasks[taskid].nodeid = nid;
     tasks[taskid].pmlid = _pid;
     tasks[taskid].prio = rcvPrio;
     tasks[taskid].preemptable = false;
     tasks[taskid].state = Ready;
     printf("@@@ %d TASK Runner\n",_pid,taskid);
     if
     :: multicore ->
          printf("@@@ %d CALL SetProcessor %d\n", _pid, rcvCore);
     :: else
     fi
     Release(startSema); // make sure stuff starts */
     /* printf("@@@ %d LOG Receiver Task %d running on Node %d\n",_pid,taskid,nid); */
     Obtain(rcvSema);

     // If the receiver is higher priority then it will be running
     // The sender is either blocked waiting for its semaphore
     // or because it is lower priority.
     // A high priority receiver needs to release the sender now, before it
     // gets blocked on its own event receive.
     if
     :: rcvPrio < sendPrio -> Release(sendSema);  // Release send semaphore for preemption
     :: else
     fi
     if
     :: doReceive ->
       printf("@@@ %d SCALAR pending %d %d\n",_pid,taskid,tasks[taskid].pending);
       if
       :: sendTwice && !sentFirst -> Release(sendSema)
       :: else
       fi
       printf("@@@ %d CALL event_receive %d %d %d %d %d recrc\n",
              _pid,rcvEvents,rcvWait,rcvAll,rcvInterval,rcvOut);
                 /* (self,  evts,     when,   what,  ticks,      out,   rc) */
       event_receive(taskid,rcvEvents,rcvWait,rcvAll,rcvInterval,rcvOut,recrc);
       printf("@@@ %d SCALAR recrc %d\n",_pid,recrc);
       if
       :: rcvOut > 0 ->
         printf("@@@ %d SCALAR recout %d %d\n",_pid,rcvOut,recout[rcvOut]);
       :: else
       fi
       printf("@@@ %d SCALAR pending %d %d\n",_pid,taskid,tasks[taskid].pending);
     :: else
     fi
     Release(sendSema);
     printf("@@@ %d LOG Receiver %d finished\n",_pid,taskid);
     tasks[taskid].state = Zombie;
     printf("@@@ %d STATE %d Zombie\n",_pid,taskid)
   }

System Process
~~~~~~~~~~~~~~

 We need a process that periodically wakes up blocked processes. This is
 modelling background behaviour of the system, such as ISRs and scheduling. We
 visit all tasks in round-robin order (to prevent starvation) and make them
 ready if waiting on other things. Tasks waiting for events or timeouts are
 not touched. This terminates when all tasks are zombies.

.. code-block:: c

   proctype System () {
     byte taskid ;
     bool liveSeen;
     printf("@@@ %d LOG System running...\n",_pid);
     loop:
     taskid = 1;
     liveSeen = false;
     printf("@@@ %d LOG Loop through tasks...\n",_pid);
     atomic {
       printf("@@@ %d LOG Scenario is ",_pid);
       printm(scenario); nl();
     }
     do   // while taskid < TASK_MAX ....
     ::  taskid == TASK_MAX -> break;
     ::  else ->
         atomic {
           printf("@@@ %d LOG Task %d state is ",_pid,taskid);
           printm(tasks[taskid].state); nl()
         }
         if
         :: tasks[taskid].state == Zombie -> taskid++
         :: else ->
            if
            ::  tasks[taskid].state == OtherWait
                -> tasks[taskid].state = Ready
                   printf("@@@ %d STATE %d Ready\n",_pid,taskid)
            ::  else -> skip
            fi
            liveSeen = true;
            taskid++
         fi
     od
     printf("@@@ %d LOG ...all visited, live:%d\n",_pid,liveSeen);
     if
     ::  liveSeen -> goto loop
     ::  else
     fi
     printf("@@@ %d LOG All are Zombies, game over.\n",_pid);
     stopclock = true;
   }

Clock Process
~~~~~~~~~~~~~

We need a process that handles a clock tick, by decrementing the tick count for
tasks waiting on a timeout. Such a task whose ticks become zero is then made
Ready, and its timer status is flagged as a timeout. This terminates when all
tasks are zombies (as signalled by ``System()`` via ``stopclock``).

.. code-block:: c

   proctype Clock () {
     int tid, tix;
     printf("@@@ %d LOG Clock Started\n",_pid)
     do
     ::  stopclock  -> goto stopped
     ::  !stopclock ->
         printf(" (tick) \n");
         tid = 1;
         do
         ::  tid == TASK_MAX -> break
         ::  else ->
             atomic{
               printf("Clock: tid=%d, state=",tid);
               printm(tasks[tid].state); nl()
             };
             if
             ::  tasks[tid].state == TimeWait ->
                 tix = tasks[tid].ticks - 1;
                 if
                 ::  tix == 0
                     tasks[tid].tout = true
                     tasks[tid].state = Ready
                     printf("@@@ %d STATE %d Ready\n",_pid,tid)
                 ::  else
                     tasks[tid].ticks = tix
                 fi
             ::  else // state != TimeWait
             fi
             tid = tid + 1
         od
     od
   stopped:
     printf("@@@ %d LOG Clock Stopped\n",_pid);
   }


init Process
~~~~~~~~~~~~

The initial process outputs annotations for defines and declarations,
generates a scenario non-deterministically and then starts the system, clock
and send and receive processes running. It then waits for those to complete,
and them, if test generation is underway, asserts ``false`` to trigger a
seach for counterexamples:

.. code-block:: c

   init {
     pid nr;
     printf("@@@ %d NAME Event_Manager_TestGen\n",_pid)
     outputDefines();
     outputDeclarations();
     printf("@@@ %d INIT\n",_pid);
     chooseScenario();
     run System();
     run Clock();
     run Sender(THIS_NODE,SEND_ID);
     run Receiver(THIS_NODE,RCV_ID);
     _nr_pr == 1;
   #ifdef TEST_GEN
     assert(false);
   #endif
   }

The information regarding when tasks should wait and/or restart
can be obtained by tracking the process identifiers,
and noting when they change.
The ``spin2test`` program does this,
and also produces separate test code segments for each Promela process.



Annotations
^^^^^^^^^^^

File: ``xxx-model.pml``

Refinement
^^^^^^^^^^

File: ``xxx-model-rfn.yml``


The test-code we generate here is based on the test-code generated from the
specification items used to describe the Event Manager in the main (non-formal)
part of the new qualification material.

The relevant specification item is ``spec/rtems/event/req/send-receive.yml``
found in ``rtems-central``. The corresponding C test code is
``tr-event-send-receive.c`` found in ``rtems`` at ``testsuites/validation``.
That automatically generated C code is a single file that uses a set of deeply
nested loops to iterate through the scenarios it generates.

Our approach is to generate a stand-alone C code file for each scenario
(``tr-event-mgr-model-N.c`` for ``N`` in range 0..8.)


The ``TASK`` annotations issued by the ``Sender`` and ``Receiver`` processes
lookup the following refinement entries, to get code that tests that the C
code Task does correspond to what is being defined in the model.

.. code-block:: yaml

   Runner: |
     checkTaskIs( ctx->runner_id );

   Worker: |
     checkTaskIs( ctx->worker_id );

The ``WAIT`` and ``SIGNAL`` annotations produced by ``Obtain()`` and
``Release()`` respectively, are mapped to the corresponding operations on
RTEMS semaphores in the test code.

.. code-block:: yaml

   code content
   SIGNAL: |
     Wakeup( semaphore[{}] );

   WAIT: |
     Wait( semaphore[{}] );

Some of the ``CALL`` annotations are used to do more complex test setup
involving priorities, or other processors and schedulers. For example:

.. code-block:: yaml

   HigherPriority: |
     SetSelfPriority( PRIO_HIGH );
     rtems_task_priority prio;
     rtems_status_code sc;
     sc = rtems_task_set_priority( RTEMS_SELF, RTEMS_CURRENT_PRIORITY, &prio );
     T_rsc_success( sc );
     T_eq_u32( prio, PRIO_HIGH );

   SetProcessor: |
     T_ge_u32( rtems_scheduler_get_processor_maximum(), 2 );
     uint32_t processor = {};
     cpu_set_t cpuset;
     CPU_ZERO(&cpuset);
     CPU_SET(processor, &cpuset);

Some handle more complicated test outcomes, such as observing context-switches:

.. code-block:: yaml

   CheckPreemption: |
     log = &ctx->thread_switch_log;
     T_eq_sz( log->header.recorded, 2 );
     T_eq_u32( log->events[ 0 ].heir, ctx->runner_id );
     T_eq_u32( log->events[ 1 ].heir, ctx->worker_id );


Most of the other refinement entries are similar to those described above for
the Chains API.

Testing Barriers
----------------

Documentation:  Barrier Manager section in the RTEMS Classic API Guide.

Model Directory: ``formal/promela/models/barriers``.

Model Name: ``barrier-mgr-model``.

The Barrier Manager is used to arrange for a number of tasks to wait on a 
designated barrier object, until either another tasks releases them, or a 
given number of tasks are waiting, at which point they are all released.

API Model
^^^^^^^^^

File: ``barrier-mgr-model.pml``

The fine details of the Promela model here, in
``barrier-mgr-model.pml``, differs from those used for the Event Manager.

Behaviour Patterns
^^^^^^^^^^^^^^^^^^

File: ``barrier-mgr-model.pml``

The overall architecture in terms of Promela  processes is very similar to the
the Event Manager with processes ``init``, ``System``, ``Clock``, ``Sender``, and ``Receiver``.

Annotations
^^^^^^^^^^^

File: ``barrier-mgr-model.pml``

Refinement
^^^^^^^^^^

File: ``barrier-mgr-model-rfn.yml``






Testing Messages
----------------

Documentation:  Message Manager section in the RTEMS Classic API Guide.

Model Directory: ``formal/promela/models/messages``.

Model Name: ``msg-mgr-model``.

The Message Manager provides objects that act as message queues. Tasks can 
interact with these by enqueuing and/or dequeuing message objects.

API Model
^^^^^^^^^

File: ``msg-mgr-model.pml``

The fine details of the Promela model here, in
``msg-mgr-model.pml``, differs from those used for the Event Manager.

Behaviour Patterns
^^^^^^^^^^^^^^^^^^

File: ``msg-mgr-model.pml``

The overall architecture in terms of Promela processes is very similar to the
Event Manager with processes ``init``, ``System``, ``Clock``, ``Sender``, and
``Receiver``.

Annotations
^^^^^^^^^^^

File: ``msg-mgr-model.pml``

Refinement
^^^^^^^^^^

File: ``msg-mgr-model-rfn.yml``

Future Work
-----------

**Discuss model re-design and having re-usable models of (1) tx-support.c facilities (e.g., simple binary semaphores), and (2) common infrastructure behaviour such as System and Clock.**


