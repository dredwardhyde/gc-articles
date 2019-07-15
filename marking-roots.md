## How does marking GC roots work in Hotspot?

In my first article I’d like to talk about some interesting solutions that have found their way into the most popular JVM implementation in the world – Hotspot JVM, originally developed by Sun Microsystems and then refined by Oracle to the current state.

Typical JVM implementation consists of multiple subsystems, like various parsers, class loaders, interpreters, JIT compilers and so on, but possibly, the most complicated subsystem is related to garbage collection. 

**First**, the only purpose of JVM runtime is to execute compatible bytecode, that is to allocate and destroy objects, invoke methods and provide system resources – and all of it – by strictly following JLS rules. 
So, runtime is all about manipulating and executing bytecode in the most effective way, while bytecode itself consists of classes and methods. In practice, you have to solve many wonderful engineering problems to manipulate it correctly. One of  challenges is to clean garbage – unreachable objects - because Java is a language with automatic memory management. Class instances - objects – small allocated regions of heap with its own header and pointers to it – are everywhere in runtime and therefore garbage collector has to know about everything that happens during bytecode execution and must be aware about various code manipulations like JIT compiling, optimizations and object's locking. So, code related to garbage collection is almost everywhere in JVM implementation and engineers have to keep that in mind while implementing new features.

**Second**, runtime environment must execute code very effectively – with minimal performance penalty and without noticeable hiccups - and even change that code in certain ways to make its execution faster, because, often enough, developers do not think about performance until it’s too late. Optimizations in garbage collections come in two ways – try not to create new garbage or eliminate it in fastest way possible. First type could be done by JIT compiler – it’s called escape analysis and it allows JIT to allocate some objects on stack instead of heap – so object will be destroyed along with enclosing method frame. But burden of second class of optimizations lies completely on the shoulders of garbage collector's solution architects and developers.

### What steps typical GC cycle consists of?

For Concurrent Mark Sweep (CMS) collector, for example, the concurrent collection cycle typically includes the following steps:

- **Stop all application threads, identify the set of objects reachable from roots, and then resume all application threads.**
- Concurrently trace the reachable object graph, using one or more processors, while the application threads are executing.
- Concurrently retrace sections of the object graph that were modified since the tracing in the previous step, using one processor.
- Stop all application threads and retrace sections of the roots and object graph that may have been modified since they were last examined, and then resume all application threads.
- Concurrently sweep up the unreachable objects to the free lists used for allocation, using one processor.
- Concurrently resize the heap and prepare the support data structures for the next collection cycle, using one processor.

Today I'm want to talk about first step of GC cycle - identifying and marking root objects. It's common step for all garbage collectors in Hotspot - Serial, Parallel, CMS, G1 and Shenandoah, and it has few peculiar implementation details that worth writing about. 

### What are roots?

Well, by quoting **Garbage Collection Handbook**, *“there are some finite set of mutator roots, representing pointers held in storage that is directly accessible to the mutator without going through other objects”*. And that’s correct definition, but there are a lot more roots than were mentioned.
In Hotspot roots are following objects:
- All JNI global references
- All inflated monitors
- All classes loaded by the boot class loader (or all classes in the event that class unloading is disabled)
- All java threads
- For each java thread then all locals and JNI local references on the thread's execution stack
- All visible/explainable objects from Universes::oops_do

Later I will explain every step in details using Parallel Garbage collector as an example because of its very simple and readable implementation.

Marking roots in Parallel GC starts from method **do_it** in **hotspot/share/gc/parallel/pcTasks.cpp**
```cpp
void MarkFromRootsTask::do_it(GCTaskManager* manager, uint which) {
  ...
  PCMarkAndPushClosure mark_and_push_closure(cm);
  ...
  switch (_root_type) {
    case universe:
      Universe::oops_do(&mark_and_push_closure);
      break;

    case jni_handles:
      JNIHandles::oops_do(&mark_and_push_closure);
      break;

    case threads:
    {
      ResourceMark rm;
      MarkingCodeBlobClosure each_active_code_blob(&mark_and_push_closure, !CodeBlobToOopClosure::FixRelocations);
      Threads::oops_do(&mark_and_push_closure, &each_active_code_blob);
    }
    break;

    case object_synchronizer:
      ObjectSynchronizer::oops_do(&mark_and_push_closure);
      break;

    case management:
      Management::oops_do(&mark_and_push_closure);
      break;

    case jvmti:
      JvmtiExport::oops_do(&mark_and_push_closure);
      break;

    case system_dictionary:
      SystemDictionary::oops_do(&mark_and_push_closure);
      break;

    case class_loader_data: {
        CLDToOopClosure cld_closure(&mark_and_push_closure, ClassLoaderData::_claim_strong);
        ClassLoaderDataGraph::always_strong_cld_do(&cld_closure);
      }
      break;

    case code_cache:
      // Do not treat nmethods as strong roots for mark/sweep, since we can unload them.
      //CodeCache::scavenge_root_nmethods_do(CodeBlobToOopClosure(&mark_and_push_closure));
      AOTLoader::oops_do(&mark_and_push_closure);
      break;

    default:
      fatal("Unknown root type");
  }
  ...
}
```

**First step** in the method above is to mark some objects in **Universe** - namespace holding known system classes and objects in the VM. Whole initial world initialization, like loading basic classes, invocation of **vmSymbols::initialize()**, **SystemDictionary::initialize()** and various necessary object allocations happens in **Universe::genesis()**.
So we need to mark some of them in **oops_do** method in **hotspot/share/memory/universe.cpp**
```cpp
void Universe::oops_do(OopClosure* f) {

  // Primitive objects
  // Here we mark all primitive type mirrors classes (like int.class, long.class, etc) 
  // created in Universe::initialize_basic_type_mirrors()
  f->do_oop((oop*) &_int_mirror);
  f->do_oop((oop*) &_float_mirror);
  f->do_oop((oop*) &_double_mirror);
  f->do_oop((oop*) &_byte_mirror);
  f->do_oop((oop*) &_bool_mirror);
  f->do_oop((oop*) &_char_mirror);
  f->do_oop((oop*) &_long_mirror);
  f->do_oop((oop*) &_short_mirror);
  f->do_oop((oop*) &_void_mirror);

  for (int i = T_BOOLEAN; i < T_VOID+1; i++) {
    f->do_oop((oop*) &_mirrors[i]);
  }
  assert(_mirrors[0] == NULL && _mirrors[T_BOOLEAN - 1] == NULL, "checking");

  // Canonicalized obj array of type java.lang.Class
  f->do_oop((oop*)&_the_empty_class_klass_array);

  // A unique object pointer unused except as a sentinel for null.
  f->do_oop((oop*)&_the_null_sentinel);

  // A cache of "null" as a Java string
  f->do_oop((oop*)&_the_null_string);

  // A cache of "-2147483648" as a Java string
  f->do_oop((oop*)&_the_min_jint_string);

  // preallocated error objects (no backtrace)
  f->do_oop((oop*)&_out_of_memory_error_java_heap);
  f->do_oop((oop*)&_out_of_memory_error_metaspace);
  f->do_oop((oop*)&_out_of_memory_error_class_metaspace);
  f->do_oop((oop*)&_out_of_memory_error_array_size);
  f->do_oop((oop*)&_out_of_memory_error_gc_overhead_limit);
  f->do_oop((oop*)&_out_of_memory_error_realloc_objects);
  f->do_oop((oop*)&_out_of_memory_error_retry);

  // preallocated cause message for delayed StackOverflowError
  f->do_oop((oop*)&_delayed_stack_overflow_error_message);

  // array of preallocated error objects with backtrace
  f->do_oop((oop*)&_preallocated_out_of_memory_error_array);

  // preallocated exception object
  f->do_oop((oop*)&_null_ptr_exception_instance);

  // preallocated exception object
  f->do_oop((oop*)&_arithmetic_exception_instance);

  // preallocated exception object
  f->do_oop((oop*)&_virtual_machine_error_instance);

  // Reference to the main thread group object
  f->do_oop((oop*)&_main_thread_group);

  // Reference to the system thread group object
  f->do_oop((oop*)&_system_thread_group);

  // The object used as an exception dummy when exceptions are thrown for
  // the vm thread.
  f->do_oop((oop*)&_vm_exception);

  // References waiting to be transferred to the ReferenceHandler
  f->do_oop((oop*)&_reference_pending_list);
  debug_only(f->do_oop((oop*)&_fullgc_alot_dummy_array);)
}
```
**Second step** is to mark references in global JNI handles:
```cpp
// hotspot/share/runtime/jniHandles.cpp
void JNIHandles::oops_do(OopClosure* f) {
  global_handles()->oops_do(f);
}
```
List of global JNI handles resides in JNIHandles class:
```cpp
class JNIHandles : AllStatic {
  friend class VMStructs;
 private:
  static OopStorage* _global_handles;
  ...
}
```
In general, **OopStorage** is container for thread-safe (sometimes concurrent) interactions with off-heap references to objects allocated in the Java heap. **\_global_handles** contains JNI handles for ArrayOutOfBoundsException, ArrayStoreException, ClassCastException, classloaders, oop wrappers used by JIT compilers, compilers threads themselves, etc. Internally every **OopStorage** contains set of **Blocks** objects, and **Block** itself contains an **oop[]** array and a bitmask indicating which entries are in use (have been allocated and not yet released). During garbage collection, collector must know about all **OopStorage** objects and their reference strength, and each **OopStorage** provides the garbage collector with support for iteration over all the allocated entries. So **oops_do()** on **OopStorage** eventually calls **iterate_impl()** method, which iterates over **Block**s in **hotspot/share/gc/shared/oopStorage.inline.hpp**:
```cpp
// Support for serial iteration, always at a safepoint.
// Provide const or non-const iteration, depending on whether Storage is
// const OopStorage* or OopStorage*, respectively.
template<typename F, typename Storage> // Storage := [const] OopStorage
inline bool OopStorage::iterate_impl(F f, Storage* storage) {
  assert_at_safepoint();
  // Propagate const/non-const iteration to the block layer, by using
  // const or non-const blocks as corresponding to Storage.
  typedef typename Conditional<IsConst<Storage>::value, const Block*, Block*>::type BlockPtr;
  ActiveArray* blocks = storage->_active_array;
  size_t limit = blocks->block_count();
  for (size_t i = 0; i < limit; ++i) {
    BlockPtr block = blocks->at(i);
    if (!block->iterate(f)) {
      return false;
    }
  }
  return true;
}
```
And then each **Block** iterates over all stored **oops** in **\_data** array:
```cpp
// Provide const or non-const iteration, depending on whether BlockPtr
// is const Block* or Block*, respectively.
template<typename F, typename BlockPtr> // BlockPtr := [const] Block*
inline bool OopStorage::Block::iterate_impl(F f, BlockPtr block) {
  uintx bitmask = block->allocated_bitmask();
  while (bitmask != 0) {
    unsigned index = count_trailing_zeros(bitmask);
    bitmask ^= block->bitmask_for_index(index);
    if (!f(block->get_pointer(index))) {
      return false;
    }
  }
  return true;
}
// Fixed-sized array of oops, plus bookkeeping data.
// All blocks are in the storage's _active_array, at the block's _active_index.
class OopStorage::Block /* No base class, to avoid messing up alignment. */ {
  ...
  oop _data[BitsPerWord];
  volatile uintx _allocated_bitmask; // One bit per _data element.
  ...
}
```
**Next step** is tricky - we need to mark all Java object references stored on Thread's stacks and internal JVM objects related to threads.
Process starts from **Threads::oops_do()** method:
```cpp
// Operations on the Threads list for GC.  These are not explicitly locked,
// but the garbage collector must provide a safe context for them to run.
// In particular, these things should never be called when the Threads_lock
// is held by some other thread. (Note: the Safepoint abstraction also
// uses the Threads_lock to guarantee this property. It also makes sure that
// all threads gets blocked when exiting or starting).

void Threads::oops_do(OopClosure* f, CodeBlobClosure* cf) {
  ALL_JAVA_THREADS(p) {
    p->oops_do(f, cf);
  }
  VMThread::vm_thread()->oops_do(f, cf);
}
```
We iterate over all Threads using *"the ugliest for loop the world has seen"*:
```cpp
// Possibly the ugliest for loop the world has seen. C++ does not allow
// multiple types in the declaration section of the for loop. In this case
// we are only dealing with pointers and hence can cast them. It looks ugly
// but macros are ugly and therefore it's fine to make things absurdly ugly.
#define DO_JAVA_THREADS(LIST, X)                                                                                          \
    for (JavaThread *MACRO_scan_interval = (JavaThread*)(uintptr_t)PrefetchScanIntervalInBytes,                           \
             *MACRO_list = (JavaThread*)(LIST),                                                                           \
             **MACRO_end = ((JavaThread**)((ThreadsList*)MACRO_list)->threads()) + ((ThreadsList*)MACRO_list)->length(),  \
             **MACRO_current_p = (JavaThread**)((ThreadsList*)MACRO_list)->threads(),                                     \
             *X = (JavaThread*)prefetch_and_load_ptr((void**)MACRO_current_p, (intx)MACRO_scan_interval);                 \
         MACRO_current_p != MACRO_end;                                                                                    \
         MACRO_current_p++,                                                                                               \
             X = (JavaThread*)prefetch_and_load_ptr((void**)MACRO_current_p, (intx)MACRO_scan_interval))

// All JavaThreads
#define ALL_JAVA_THREADS(X) DO_JAVA_THREADS(ThreadsSMRSupport::get_java_thread_list(), X)
```
and call **JavaThread::oops_do(OopClosure\* f, CodeBlobClosure\* cf)**:
```cpp
void JavaThread::oops_do(OopClosure* f, CodeBlobClosure* cf) {
  ...
  // Traverse the GCHandles
  Thread::oops_do(f, cf);
  ...
  if (has_last_Java_frame()) {
    ...
    // traverse the registered growable array - list of the protection domains on the current execution stack
    if (_array_for_gc != NULL) {
      for (int index = 0; index < _array_for_gc->length(); index++) {
        f->do_oop(_array_for_gc->adr_at(index));
      }
    }
    // Traverse the monitor chunks - off stack monitors allocated 
    // during deoptimization and by JNI_MonitorEnter/Exit
    for (MonitorChunk* chunk = monitor_chunks(); chunk != NULL; chunk = chunk->next()) {
      chunk->oops_do(f);
    }
    // Traverse the execution stack
    for (StackFrameStream fst(this); !fst.is_done(); fst.next()) {
      fst.current()->oops_do(f, cf, fst.register_map());
    }
  }
  ...
  GrowableArray<jvmtiDeferredLocalVariableSet*>* list = deferred_locals();
  if (list != NULL) {
    for (int i = 0; i < list->length(); i++) {
      list->at(i)->oops_do(f);
    }
  }
  // Traverse instance variables at the end since the GC may be moving things
  // around using this function
  f->do_oop((oop*) &_threadObj);
  f->do_oop((oop*) &_vm_result);
  f->do_oop((oop*) &_exception_oop);
  f->do_oop((oop*) &_pending_async_exception);

  if (jvmti_thread_state() != NULL) {
    jvmti_thread_state()->oops_do(f);
  }
}
```
In **Thread::oops_do()** we mark all active JNI handles, possible **\_pending_exception**, all thread-local handles in **HandleArea\* \_handle_area**, reserved for allocation of handles within the VM and thread-local monitors:
```cpp
void Thread::oops_do(OopClosure* f, CodeBlobClosure* cf) {
  active_handles()->oops_do(f);
  // Do oop for ThreadShadow
  f->do_oop((oop*)&_pending_exception);
  handle_area()->oops_do(f);

  // We scan thread local monitor lists here, and the remaining global
  // monitors in ObjectSynchronizer::oops_do().
  ObjectSynchronizer::thread_local_used_oops_do(this, f);
}
```
In **active_handles()->oops_do(f)** we traverse and mark all active JNI handles for each thread, because every **JNIHandleBlock** in **JNIHandleBlock\* \_active_handles** contains strong roots - references to Java objects that were passed as parameters to JNI methods:
```cpp
class JNIHandleBlock : public CHeapObj<mtInternal> {
  ...
 private:
  ...
  oop             _handles[block_size_in_oops]; // The handles to Java objects allocated in heap
}
```
JNI methods are unmanaged area, so objects passed as parameters are considered alive, we need to mark them:
```cpp
void JNIHandleBlock::oops_do(OopClosure* f) {
  JNIHandleBlock* current_chain = this;
  // Iterate over chain of blocks, followed by chains linked through the
  // pop frame links.
  while (current_chain != NULL) {
    for (JNIHandleBlock* current = current_chain; current != NULL;
         current = current->_next) { // Link to next block
      ...
      for (int index = 0; index < current->_top; index++) {
        oop* root = &(current->_handles)[index];
        oop value = *root;
        // traverse heap pointers only, not deleted handles or free list
        // pointers
        if (value != NULL && Universe::heap()->is_in_reserved(value)) {
          f->do_oop(root);
        }
      }
      // the next handle block is valid only if current block is full
      if (current->_top < block_size_in_oops) {
        break;
      }
    }
    current_chain = current_chain->pop_frame_link();
  }
}
```    
Also each thread could hold arbitrary number of local inflated locks - **ObjectMonitor** objects. All of them must be marked by **ObjectSynchronizer::thread_local_used_oops_do()** method because they contain backward object pointers:
```cpp
class ObjectMonitor {
 ...
 private:
  ...
  void*     volatile _object;       // backward object pointer - strong root
```
Next major step in **JavaThread::oops_do()** is traversal of execution stack of current thread. What is execution stack? It's a group of frames, some of them are from native methods (like JIT compiled), some - from interpretered, and others are just StubRoutine entries. And every such frame contains local variables and passed parameters that act like strong roots during garbage collection. So we need to identify them and mark. Marking roots in **frame** starts in **frame::oops_do_internal()** method: 
```cpp
void frame::oops_do_internal(OopClosure* f, CodeBlobClosure* cf, RegisterMap* map, bool use_interpreter_oop_map_cache) {
  ...
  //interpreted frame
  if (is_interpreted_frame()) {
    oops_interpreted_do(f, map, use_interpreter_oop_map_cache);
    // entry frame for one of StubRoutines defined in hotspot/share/runtime/stubRoutines.cpp
    // StubRoutines provides entry points to assembly routines used by
    // compiled code and the run-time system. Platform-specific entry
    // points are defined in the platform-specific inner class.
  } else if (is_entry_frame()) {
    oops_entry_do(f, map);
  } else if (CodeCache::contains(pc())) {
    oops_code_blob_do(f, cf, map);
  } else {
    ShouldNotReachHere();
  }
}
```
If frame is **StubRoutine** entry, then we need to mark passed arguments, the receiver of the call (if a non-static call) and objects saved in **JNIHandleBlock** blocks:
```cpp
// hotspot/share/runtime/stubRoutines.cpp
class JavaCallWrapper: StackObj {
  ...
 private:
  JavaThread*      _thread;                 // the thread to which this call belongs
  JNIHandleBlock*  _handles;                // the saved handle block
  Method*          _callee_method;          // to be able to collect arguments if entry frame is top frame
  oop              _receiver;               // the receiver of the call (if a non-static call)
}

// hotspot/share/runtime/frame.cpp
void frame::oops_entry_do(OopClosure* f, const RegisterMap* map) {
  ...
  if (map->include_argument_oops()) {
    // must collect argument oops, as nobody else is doing it
    Thread *thread = Thread::current();
    methodHandle m (thread, entry_frame_call_wrapper()->callee_method());
    EntryFrameOopFinder finder(this, m->signature(), m->is_static());
    finder.arguments_do(f);
  }
  // Traverse the Handle Block saved in the entry frame
  entry_frame_call_wrapper()->oops_do(f);
}
```
If frame is interpreted, then **frame::oops_interpreted_do()** method is called. Here we mark **BasicObjectLock** objects because they contain references to Java objects, just like inflated locks. And then we need to mark all local references inside the current frame. That's where **OopMapCache** is used - this cache holds location of object references in an interpreted frame:
```cpp
void frame::oops_interpreted_do(OopClosure* f, const RegisterMap* map, bool query_oop_map_cache) {
  ...
  Thread *thread = Thread::current();
  methodHandle m (thread, interpreter_frame_method());
  jint      bci = interpreter_frame_bci();
  ...
  // Handle the monitor elements in the activation
  for ( BasicObjectLock* current = interpreter_frame_monitor_end();
        current < interpreter_frame_monitor_begin();
        current = next_monitor_in_interpreter_frame(current)) {
    ...
    current->oops_do(f);
  }
  //
  if (m->is_native()) {
    f->do_oop(interpreter_frame_temp_oop_addr());
  }

  // The method pointer in the frame might be the only path to the method's
  // klass, and the klass needs to be kept alive while executing.
  f->do_oop(interpreter_frame_mirror_addr());

  // Process a callee's arguments if we are at a call site (i.e., if we are at an invoke bytecode)
  ... marking arguments here ...
  ...
  // process locals & expression stack
  InterpreterOopMap mask;
  if (query_oop_map_cache) {
    m->mask_for(bci, &mask);
  } else {
    OopMapCache::compute_one_oop_map(m, bci, &mask);
  }
  ...
}
```
There is only one instance of **OopMapCache** per **InstanceKlass** (VM level representation of a Java class) and it must be allocated lazily at first request. Then it will be queried and populated in **OopMapCache::lookup()** method. But there is a subtle detail in lookup of **OopMapCache** - it happens at the **global safepoint**. It means average depth of execution stacks is dramatically influence duration of stop-the-world pause caused by root marking. That schema is used even in the most modern garbage collectors, like G1 and Shenandoah.
```cpp
// Called by GC for thread root scan during a safepoint only.  The other interpreted frame oopmaps
// are generated locally and not cached.
void OopMapCache::lookup(const methodHandle& method, int bci, InterpreterOopMap* entry_for) {
  assert(SafepointSynchronize::is_at_safepoint(), "called by GC in a safepoint");
  ...
}
```
For compiled frames we have a little different set of tools. First, **CodeCache** holds various pieces of generated code - compiled java methods, runtime stubs, transition frames, etc. The entries in the CodeCache are all CodeBlob's. **CodeCache** could be updated, for example, after compilation by C1 in **Compilation::install_code()** method or during deoptimization event. So both profiled and non-profiled JIT-generated method (nmethods) are stored here. By checking **CodeCache::contains(pc())** in **frame::oops_do_internal()** we could tell if it is native method or not and process it appropriately in **frame::oops_code_blob_do()**:
```cpp
class frame {
 private:
  ...
  CodeBlob* _cb; // CodeBlob that "owns" pc
}

void frame::oops_code_blob_do(OopClosure* f, CodeBlobClosure* cf, const RegisterMap* reg_map) {
  if (_cb->oop_maps() != NULL) {
    OopMapSet::oops_do(this, reg_map, f);
    // Preserve potential arguments for a callee. We handle this by dispatching
    // on the codeblob. For c2i, we do
    if (reg_map->include_argument_oops()) {
      _cb->preserve_callee_argument_oops(*this, reg_map, f);
    }
  }
  ...
  if (cf != NULL)
    cf->do_code_blob(_cb);
}
```
Each **CodeBlob** holds **ImmutableOopMapSet\* \_oop_maps** - set of **OopMap** entries. **OopMapValue** represents a single **OopMap** entry and describes for a specific pc whether each register and frame stack slot is a reference to Java object or not. If it is a reference  (**oop**) then we have to mark it as strong root, just as usual.

**Next step** of roots marking in **MarkFromRootsTask::do_it()** is to mark all global **ObjectMonitor** objects by invoking **ObjectSynchronizer::oops_do**. This process also happens at the global safepoint.

In **Management::oops_do()** we mark all roots stored in **MemoryService** and **ThreadService**. **MemoryService** provides VM-side monitoring and management support and holds references to **MemoryPool** and **MemoryManager** instances. Each **MemoryPool** holds references to **Sensor** Java objects - **\_usage_sensor** and **\_gc_usage_sensor** passed from **MemoryPoolImpl.setUsageThreshold()** method. **ThreadService** contains **ThreadSnapshot** objects with references to Java Thread. All the objects described above must be also marked as strong roots.

In **JvmtiExport::oops_do()** we mark all Java objects which references are transitively stored in **JvmtiBreakpointCache** and allocated directly from JVM TI. 

**Next important step** of roots marking process is to identify all strong roots referenced from **SystemDictionary**.
In **SystemDictionary::oops_do()** we mark VM-side mirrors of system and platform **ClassLoader**s, etc. See comments in code snippet below:
```cpp
void SystemDictionary::oops_do(OopClosure* f) {
  // System ClassLoader
  f->do_oop(&_java_system_loader);
  // Platform ClassLoader
  f->do_oop(&_java_platform_loader);
  // Lock object for system class loader
  f->do_oop(&_system_loader_lock_obj);
  CDS_ONLY(SystemDictionaryShared::oops_do(f);)

  // Visit extra methods
  invoke_method_table()->oops_do(f);
}
void SystemDictionaryShared::oops_do(OopClosure* f) {
  // These _shared_xxxs arrays are used to initialize the java.lang.Package and
  // java.security.ProtectionDomain objects associated with each shared class.
  f->do_oop((oop*)&_shared_protection_domains);
  f->do_oop((oop*)&_shared_jar_urls);
  f->do_oop((oop*)&_shared_jar_manifests);
}
```
**invoke_method_table()** is called on **SymbolPropertyTable\* \_invoke_method_table**. **SymbolPropertyTable** is a system-internal mapping of symbols to pointers. For example, this table holds references to low-level intrinsic methods defined by  JVM. For example, entry to **SymbolPropertyTable** could be added in **SystemDictionary::find_method_handle_intrinsic(vmIntrinsics::ID iid, Symbol\* signature)** method during polymorphic method lookup at call site.

**ClassLoaderDataGraph::always_strong_cld_do(&cld_closure)** marks all Java objects references from **Chunk**s. Those objects are instances of java/lang/ClassLoader associated with current **ClassLoaderData**, constant pool arrays, Modules, etc. All objects have the same life cycle of the corresponding ClassLoader.
```cpp
void ClassLoaderDataGraph::always_strong_cld_do(CLDClosure* cl) {
  assert_locked_or_safepoint_weak(ClassLoaderDataGraph_lock);
  if (ClassUnloading) {
    roots_cld_do(cl, NULL);
  } else {
    cld_do(cl);
  }
}
void ClassLoaderDataGraph::roots_cld_do(CLDClosure* strong, CLDClosure* weak) {
  ...
  for (ClassLoaderData* cld = _head;  cld != NULL; cld = cld->_next) {
    CLDClosure* closure = cld->keep_alive() ? strong : weak;
    if (closure != NULL) {
      closure->do_cld(cld);
    }
  }
}
void ClassLoaderData::oops_do(OopClosure* f, int claim_value, bool clear_mod_oops) {
  ...
  ChunkedHandleList _handles.oops_do(f);
}
inline void ClassLoaderData::ChunkedHandleList::oops_do_chunk(OopClosure* f, Chunk* c, const juint size) {
  for (juint i = 0; i < size; i++) {
    if (c->_data[i] != NULL) {
      // mark contents of _data array of each Chunk in ChunkedHandleList
      f->do_oop(&c->_data[i]);
    }
  }
}
struct Chunk : public CHeapObj<mtClass> {
      ...
      oop _data[CAPACITY];
      ...
      Chunk* _next;
      ...
    };
```   
And as the final step of marking roots, **AOTLoader::oops_do()** must mark all referenced Java objects. **AOTLoader** is used for ahead-of-time compilation that was added in [JEP 295](https://openjdk.java.net/jeps/295). **AOTCompiledMethod** objects hold references to enclosing class and **AOTCodeHeap** contains Java object pointers **\_oop\_got**, patched by Hotspot.
```cpp
void AOTCodeHeap::oops_do(OopClosure* f) {
  for (int i = 0; i < _oop_got_size; i++) {
    oop* p = &_oop_got[i];
    if (*p == NULL)  continue;  // skip non-oops
    f->do_oop(p);
  }
  for (int index = 0; index < _method_count; index++) {
    if (_code_to_aot[index]._state != in_use) {
      continue; // Skip uninitialized entries.
    }
    AOTCompiledMethod* aot = _code_to_aot[index]._aot;
    aot->do_oops(f);
  }
}
```

## Conclusion

Even the beginning of GC cycle - process of marking roots - is quite complicated. Most of the time, people prefer to barely mention this fact while talking about garbage collectors in Hotspot. Its designers and implementors decided to make some steps described above at global safepoint. And *duration* of this steps (and so is safepoint, without additional steps not related to garbage collection) greatly depends on your code style and structure of your application. More threads and higher average depth of execution stacks, more local variables - will make global safepoint considerably longer.  
In enterprise deployments, those numbers are easily achievable - all applications on a server like Wildfly share same heap and could have up to 1000 threads with average depth of 20-40 frames.

For example, on the following OpenJDK:
```sh
openjdk version "1.8.0-internal"
OpenJDK Runtime Environment (build 1.8.0-internal-_2019_07_14_21_46-b00)
OpenJDK 64-Bit Server VM (build 25.71-b00, mixed mode)
```
and JVM options:
```sh
-XX:+UseShenandoahGC -verbose:gc -Xms512m -Xmx2048m
```
And on single Wildfly 10.1 instance with 4 idle applications running - 304 Java threads total  
and average execution stack depth around 20 frames, I've got quite interesting results:  
  
Trigger: Time since last GC (300001 ms) is larger than guaranteed interval (300000 ms)  
\[Concurrent reset 1222M->1222M(1678M), 2.952 ms]  
**\[Pause Init Mark, 14.636 ms]**  
\[Concurrent marking 1222M->1222M(1678M), 102.393 ms]  
**\[Pause Final Mark, 3.152 ms]**  
\[Concurrent cleanup 1222M->852M(1678M), 0.083 ms]  
\[Concurrent evacuation 852M->853M(1678M), 1.103 ms]  
**\[Pause Init Update Refs, 0.034 ms]**  
\[Concurrent update references 853M->853M(1678M), 38.370 ms]  
**\[Pause Final Update Refs, 2.226 ms]**  
\[Concurrent cleanup 853M->481M(1678M), 0.073 ms]  
   
Trigger: Time since last GC (300001 ms) is larger than guaranteed interval (300000 ms)  
\[Concurrent reset 1242M->1242M(1257M), 3.026 ms]  
**\[Pause Init Mark, 12.033 ms]**  
\[Concurrent marking 1242M->1242M(1257M), 104.882 ms]  
**\[Pause Final Mark, 3.100 ms]**  
\[Concurrent cleanup 1243M->871M(1257M), 0.087 ms]  
\[Concurrent evacuation 871M->873M(1259M), 1.309 ms]  
**\[Pause Init Update Refs, 0.036 ms]**  
\[Concurrent update references 873M->873M(1259M), 46.671 ms]  
**\[Pause Final Update Refs, 2.208 ms]**  
\[Concurrent cleanup 873M->485M(1259M), 0.046 ms]  
  
As you can see, **Init Mark pause** (which includes process of marking roots) is predominant pause here. 
