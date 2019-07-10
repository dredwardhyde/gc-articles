## How does marking GC roots works in Hotspot?

In this article I’d like to talk a little about some interesting solutions that have found their way into the most popular JVM implementation in the world – Hotspot JVM, originally developed by Sun Microsystems and then refined by Oracle to the current state.

Typical JVM implementation consists of multiple subsystems, like various parsers, class loaders, interpreters, JIT compilers and so on, but possibly, the most complicated subsystem is related to garbage collection. 

**First**, the only purpose of JVM runtime is to execute compatible bytecode, that is to allocate and destroy objects, invoke methods and provide system resources – and all of it – by strictly following JLS rules. 
So, runtime is all about manipulating and executing bytecode in the most effective way, and bytecode itself consists of classes and methods. In practice, you have to solve many wonderful engineering problems to manipulate them correctly. One of such challenges – is to clean garbage – unreachable objects, because we all remember that Java is a language with automatic memory management. Class instances - objects – small allocated regions of heap with header and pointer to it – are everywhere in runtime and therefore garbage collector has to know about everything that happens during bytecode execution and must be aware about various code manipulations like JIT compiling and optimizations. So, code related to garbage collection is almost everywhere in JVM implementation and engineers have to keep that in mind while implementing new features. 

**Second**, runtime environment must execute code very effectively – with minimal overhead - and even change that code in certain ways to make its execution faster, because, often enough, developers do not think about performance until it’s too late. Optimizations in garbage collections come in two ways – try not to create new garbage or eliminate it in fastest way possible. First type could be done by JIT compiler – it’s called escape analysis and it allows JIT to allocate some objects on stack instead of heap – so object will be destroyed along with enclosing method frame. But burden of second class of optimizations lies completely on the shoulders of garbage collector's solution architects and developers.

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

Later I will explain every step in details using Parallel Garbage collector as an example because of very simple and readable implementation.

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

First step in the method above is to mark some objects in **Universe** - namespace holding known system classes and objects in the VM. Whole initial world initialization, like loading basic classes, **vmSymbols::initialize()**, **SystemDictionary::initialize()** and various necessary object allocations happens in **Universe::genesis()**.
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
Second step is to mark global JNI references by JNI handles in **hotspot/share/runtime/jniHandles.cpp**.
```cpp
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
In general, **OopStorage** is container for thread-safe (sometimes concurrent) interactions with off-heap references to objects allocated in the Java heap. The garbage collector must know about all OopStorage objects and their reference strength. OopStorage provides the garbage collector with support for iteration over all the allocated entries.
And every **OopStorage** internally contains set of **Blocks** objects, and **Block** itself contains an **oop[]** and a bitmask indicating which entries are in use (have been allocated and not yet released).
**oops_do()** on **OopStorage** eventually calls **iterate_impl()** method, which iterates over **Block**:
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
And then with each **Block** iterates over all stored **oops** in **_data** array:
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
```
```cpp
// Fixed-sized array of oops, plus bookkeeping data.
// All blocks are in the storage's _active_array, at the block's _active_index.
// Non-full blocks are in the storage's _allocation_list, linked through the
// block's _allocation_list_entry.  Empty blocks are at the end of that list.
class OopStorage::Block /* No base class, to avoid messing up alignment. */ {
  // _data must be the first non-static data member, for alignment.
  oop _data[BitsPerWord];
  static const unsigned _data_pos = 0; // Position of _data.

  volatile uintx _allocated_bitmask; // One bit per _data element.
  ...
}
```









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
JNIHandleBlock contains strong roots - references to Java objects that were passed as parameters to JNI methods.
JNI method is unmanaged area, so those objects are considered alive:

```cpp
class JNIHandleBlock : public CHeapObj<mtInternal> {
  ...
 private:
  ...
  oop             _handles[block_size_in_oops]; // The handles to Java objects allocated in heap
}
```
