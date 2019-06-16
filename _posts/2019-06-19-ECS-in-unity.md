---
layout: post
title: ECS in unity
tags: [dev]

---



话说我在2015年的时候首次接触ECS 的概念，当初还写了一个简单的ECS framework和一批博客文章来记录，https://haolly.com/2015/09/10/entity-system/  ，ECS 的核心理念很简单，但是如何把它用在工程中却是另一个问题了。

# burst compiler

That said, when working on a piece of performance critical code, we can give up on most of the standard library, (bye Linq, StringFormatter, List, Dictionary), disallow allocations (=no classes, only structs), reflection, the garbage collector and virtual calls, and add a few new containers that you are allowed to use (NativeArray and friends).

# 怎么解决多线程问题（ race condition, deadlocks, nondeterminism)
job system

# Blittable types
a job can only access blittable data types
problem: The drawback to the safety system’s process of copying data is that it also isolates the results of a job within each copy. 
solution: A NativeContainer is a managed value type that provides a relatively safe C# wrapper for native memory. It contains a pointer to an unmanaged allocation. When used with the Unity C# Job System, a NativeContainer allows a job to access data shared with the main thread rather than working with a copy.

problem: By default, when a job has access to a NativeContainer, it has both read and write access. This configuration can slow performance. The C# Job System does not allow you to schedule a job that has write access to a NativeContainer at the same time as another job that is writing to it.

solution:
If a job does not need to write to a NativeContainer, mark the NativeContainer with the [ReadOnly] attribute, like so:
[ReadOnly]
public NativeArray<int> input;


# Create Job

Caveats:
1. Schedule and Complete method can only be called in the main thread
2. When designing your job, remember that they operate on copies of data, except in the case of NativeContainer. So, the only way to access data from a job in the main thread is by writing to a NativeContainer.
3. Calling Complete on a JobHandle returns ownership of that job’s NativeContainer types to the main thread. You need to call Complete on a JobHandle to safely access those NativeContainer types from the main thread again 
4. Jobs do not start executing when you schedule them. Use JobHandle.Complete  or JobHandle.ScheduleBatchedJobs to start executing them.
5. Not flushing the batch delays the scheduling until the main thread waits for the result. In all other cases use JobHandle.Complete to start the execution process. Note: In the Entity Component System (ECS) the batch is implicitly flushed for you, so calling JobHandle.ScheduleBatchedJobs is not necessary.
6. 不能直接修改NativeContainer中的值，必须借助临时变量
7. Do not allocate managed memory in jobs

# questions:
1. 多个job 都引用同一份 NativeContainer，那么这个NativeContainer 是不是也拷贝了一份呢？ 还是说只有一份？
answer: All copies of the NativeArray point to the same memory

# cache friendly
ECS
问题，不同component 的数据分散在不同的地方，因为是引用类型
解决办法，将具有相同 component 集合的 entity 放到一起，这样的一个组合叫 archetype

internal:
component 放struct类型
辅助设施，archetype query
each archetype has a list of Chunks where entities of that archetype are stored.

bonus:
use job system to run code within each entities


# What is ECS ?
entity is just a 32-bit integer, system provide behavior, components store the data

## Convert GameObject to Entity
`ConvertToEntity` MonoBehaviour, also convert build-in MonoBehaviour to Component

## Instantiate a Entity
method one: `GameObjectConversionUtility.ConvertGameObjectHierarcy(prefab, World.Active)` to convert GameObject to entity, then use
`entityManger.Instantiate` to Instantiate it
method two: implement `IDeclareReferencedPrefabs` interface when use `IConvertGameObjectToEntity` and convert it to an entity, then use
`CommandBuffer.Instantiate(prefabEntity)` to instantiate it

## Destroy Entity
`EntityCommandBuffer.DestroyEntity()`

# Hybird ECS

## Convert MonoBehaviour to Component
implement `IConvertGameObjectToEntity` in old MonoBehaviour, need `RequiresEntityConversion` Attribute

## Custom Component, struct not class
implement `IComponentData` or
implement `ISharedComponentData`

what if I need share some data between two entities:
goto `ISharedComponentData`

## Unique combination of component called **Archetype**, store in **Chunks**

## Associate entity with component
in `IConvertGameObjectToEntity` override function
```c#
public void Convert(Entity entity, EntityManager dstManager, GameObjectConversionSystem conversionSystem)
    {
        var spawnerData = new Spawner_FromEntity
        {
            // The referenced prefab will be converted due to DeclareReferencedPrefabs.
            // So here we simply map the game object to an entity reference to that prefab.
            Prefab = conversionSystem.GetPrimaryEntity(Prefab),
            CountX = CountX,
            CountY = CountY
        };
        dstManager.AddComponentData(entity, spawnerData);
    }
```

# Custom System
implement `ComponentSystem`, override `OnUpdate` function
implement `JobComponentSystem`, code sampler:
```c#
    struct RotationSpeedJob : IJobForEach<Rotation, RotationSpeed_IJobForEach>
    {
        public float DeltaTime;

        // The [ReadOnly] attribute tells the job scheduler that this job will not write to rotSpeedIJobForEach
        public void Execute(ref Rotation rotation, [ReadOnly] ref RotationSpeed_IJobForEach rotSpeedIJobForEach)
        {
            // Rotate something about its up vector at the speed given by RotationSpeed_IJobForEach.
            rotation.Value = math.mul(math.normalize(rotation.Value), quaternion.AxisAngle(math.up(), rotSpeedIJobForEach.RadiansPerSecond * DeltaTime));
        }
    }

     // Any previously scheduled jobs reading/writing from Rotation or writing to RotationSpeed 
    // will automatically be included in the inputDeps dependency.
    protected override JobHandle OnUpdate(JobHandle inputDependencies)
    {
        var job = new RotationSpeedJob
        {
            DeltaTime = Time.deltaTime
        };

        return job.Schedule(this, inputDependencies);
    }
```


## Question
how to solve race condition?
根据读写信息，自动管理依赖

## Entity is aranged in chunks, what if I need change archetype of an entity at runtime?
normal approach with EntityManager will invalide the chunk array
在一个job里面去修改一个entity的component会导致race condition，因为可能有其他的job运行在另外的core上面

so, use `EntityCommandBuffer`
EntityCommandBuffer let you queue up changes from either a job for from the main thread so that they can take effect later on the main thread
#TODO 最晦涩的第一点

例如，一个MoveUp 的component，在我按了空格键之后不再向上移动，而是向下移动，这时候有两种处理方法，第一种，改变System 处理
MoveUp的逻辑，第二种移除MoveUp 新增 MoveDown
#TODO


# System 负责更新Entity 的component， 那么怎么找到需要更新的component 呢？ 类似于 GetComponent<T>()
Entities.ForEach() , run in main thread
IJobForEach<T,..>
raw EneityQuery
etc


## 不同 System 之间的update顺序是怎么样的？ 类似于原来的 physics->input->game logic ->animation->rendering
不同的ComponentSystemGroup

#TODO, 好像不重要，对于 CommandBuffer 来说比较重要？


# 不同 System 直接怎么交互？
不同 System 之间为啥要交互， 因为Data是独立的，所以system之间不会有交互

system always run in the main thread

# 问题
1. SpawnFromMonoBehaviour 例子中，cube的旋转是谁控制的？ cube 上面只有Component 呀
还是`RotationSpeedSystem_IJobForEach`。
答：只要这个system 的代码还是，就一直起作用，虽然看不到任何引用他的地方。。。
2. 每一个system中的job 必须在一帧之内运行完？


# lua 的适用性
https://www.zhihu.com/question/286963885





# ref
https://en.wikipedia.org/wiki/Blittable_types
https://docs.unity3d.com/Manual/JobSystemTroubleshooting.html
https://docs.unity3d.com/Packages/com.unity.entities@0.0/manual/chunk_iteration.html
https://github.com/davidpol/SurvivalShooterECS
https://medium.com/@gamevanilla/survival-shooter-using-unitys-entity-component-system-revisited-874cd69085ae


