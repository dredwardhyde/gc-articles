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

