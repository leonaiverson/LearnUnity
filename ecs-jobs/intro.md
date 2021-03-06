Unity version 2018.1 introduces a few major new features for achieving high performance:

- The **[Job System](jobs.md)** farms units of work called 'jobs' out to threads while helping us maintain thread safety.
- The **Burst compiler** optimizes code using [SIMD instructions](https://en.wikipedia.org/wiki/SIMD), which is particularly beneficial for math-heavy code. The Burst compiler is not a general-purpose C# compiler: it only works on job code, which is written in a subset of C# called HPC# (High Performance C#).
- **[ECS (Entity Component System)](ecs.md)** is an architectural pattern in which we lay out data in native (non-garbage collected) memory in the optimal, linear fashion: tightly packed, contiguous, and accessible in sequence. By separating code from data, ECS also (arguably) improves code structure over the traditional Object-Oriented approach.

ECS and the Job System can be used separately, but they are [highly complementary](ecs_jobs.md): ECS guarantees data is layed out linearly in memory, which speeds up job code that accesses that data and gives the Burst compiler more optimization opportunities.

### Videos

- [the Job System](https://www.youtube.com/watch?v=zkVYbcSlfoE)
- [ECS](https://www.youtube.com/watch?v=kk8RCwQHIy4)
- [using the Job System with ECS](https://www.youtube.com/watch?v=SZGRtQ7-ilo)
- [ECS fixed arrays and shared components](https://youtu.be/oO2yqVQwFUQ)
- [ECS transforms and rendering](https://www.youtube.com/watch?v=QD2DpeuOrS0)

**TIP**: If you find the narration a bit too fast, you can set Youtube video playback to any speed you like in the Javascript console. For example, you can set the playback rate to 92% with `document.getElementsByTagName('video')[0].playbackRate = 0.92`

## Job System overview

### job execution

A job can only be *scheduled* (added to the job queue) from the main thread, but a job usually executes on one of Unity's background worker threads rather than the main thread.

When a worker thread is available, the job system executes a job waiting on the queue. The execution order of jobs on the queue is left up to the job system and is not necessarily the same as the order the jobs were added to the queue. Once started, a job runs on its thread without interuption until finished.

When we call the *complete()* method on a scheduled job, the main thread will wait for the job to finish executing (if it hasn't finished already), and all references to the job will be removed from the job system. We should complete all jobs at some point after scheduling them. 

Completing a job *immediately* after scheduling it is generally undesirable because the main thread can't do anything else while waiting for a job to complete, including schedule other jobs! So usually we schedule jobs early in the frame and complete them at the end of the frame or in a later frame.

### job input and output

A job is passed a struct as input. This struct cannot contain any memory references, except it can have NativeContainer types (such as NativeArray or NativeHashMap), which have unsafe pointers to native memory. A NativeContainer is manually allocated and so should be manually deallocated by calling its *Dispose()* method when you no longer need it.

A job only produces output by mutating the contents of NativeContainer(s) passed in the input struct. (Mutations to the input struct itself are not visible outside the job because the job gets its own private copy of the struct.) A job should not touch static fields or methods and should not do I/O, so the purpose of a job is always just to mutate the contents of one or more NativeContainers passed in *via* the input struct. (As discussed later, jobs can also mutate ECS entities and components.)

### job dependencies

When scheduling a job, we can specify another already scheduled job as its dependency. The job system will not start executing a scheduled job until that job's dependency has finished executing. This is useful when two jobs use the same NativeContainer(s) because we usually want to guarantee that one job finishes using the NativeContainer(s) before the other job starts.

Effectively, scheduled jobs can form chains of dependency. For example, if job A depends upon job B which depends upon job C, then A will not start until B has finished, and B will not start until C has finished.

A job can be the direct dependency of multiple other jobs, and a job can have multiple direct dependencies. A job that is the dependency of multiple other jobs will finish before those other jobs will start executing. A job with multiple dependencies will not start executing until all of its dependencies have finished.

Cycles of dependency (such as A depending upon B which depends back on A) are not possible because new jobs can only specify already scheduled jobs as dependencies and because we cannot change the dependencies of an already scheduled job.

Completing a job will implicitly complete all of its dependencies too.

### safety checks

It's generally a mistake to have two or more jobs concurrently use the same NativeContainer, and so, when executing our game inside the editor, Unity throws exceptions when it detects where such cases might arise. (Because these checks can be costly, they are not enabled when running your game outside the editor.)

When two jobs access the same NativeContainer, we should avoid these exceptions by either:

- completing one job before scheduling the other
- or making one job the direct or indirect dependency of the other

Either of these arrangements guarantees that one job finishes executing before the other starts. (Which of the two jobs should run first is up to us because the choice depends upon the particular logic of what we're trying to accomplish!)

## ECS (Entity Component System) overview

An **entity** is a piece of data known by a unique ID number and which can logically contain any number of **components** (not to be confused with Unity's GameObject Components). These components:

- are struct types
- can contain other value types
- cannot contain memory references (even pointers to NativeContainers)
- can reference entities by storing their unique ID numbers
- should generally be small (on the order of a few tens or hundreds of bytes)

A single entity cannot have multiple components of the same type.

A **system** is a class whose *Update()* method is called once every frame in the main thread. By default, the order of system updates within a frame is chosen automatically, but we can specify their relative execution order. A typical system update iterates through a selection of component types for all entities which have components of those types. For example, a system might iterate through all components of types A, C, and D for all entities which have components of those types (regardless of what other types of components those entities might have, so in this case, say, an entity with types A, B, C, D, and E would be included). The entities and their components are stored in memory in a linear fashion, making these iterations through the entities and their components as optimal as possible.

We can create and schedule jobs which read and mutate entities and their components, but doing so requires special consideration to avoid conflicting reads/writes between overlapping jobs. Safety checks catch these conflicts, and a special type of system called JobComponentSystem helps us structure our jobs' dependencies to avoid these conflicts.

### hybrid ECS

As of yet, Unity provides very few stock component types and systems, so a game that uses only ECS rather than GameObjects will have to implement most pieces of game functionality for itself. For example, there are no ECS components or systems yet for collision detection. Until these missing pieces are filled in over the next few years, most projects using ECS will want to use the old GameObjects as well. Just be clear that the old GameObjects do not have the performance benefits of ECS's linear memory storage, nor are GameObjects integrated into the Job System.

[next \>\>](jobs.md)
