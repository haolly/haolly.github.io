---
layout: post
title: Bullet 物理引擎在项目中的使用
tags: [gamedev]
---

## 一个 Bug

前几天项目中遇到一个 Bug，小飞机飞到大 Boss 下面后发射子弹，打不中大 Boss。

刚开始以为是 Bullet 物理引擎的问题，小的碰撞盒进入到大的碰撞盒后产生不了碰撞，于是 Google 了半天，也没找到相关的信息，第二天修改了下程序以方便单步调试，才发现原来是逻辑上的问题，这种情况下，碰撞是在子弹加入到物理引擎世界中后的那一帧就马上产生的，而子弹的表现做了延迟一帧的创建，最终在物理引擎的 **Step** 之后才去创建。于是在碰撞函数中检测到没有这个子弹的表现 ID，函数直接返回了。

趁机总结下 Bullet 这个物理引擎在飞机项目中的使用情况

## 为啥要使用 Bullet 
因为飞机的碰撞检测都需要在服务器端处理，但是 Unity 自带的物理引擎没法再服务器端使用

## Bullet Physics 是啥

### 编译
http://bulletphysics.org/mediawiki-1.5.8/index.php/Installation

### HelloWorld
* http://bulletphysics.org/mediawiki-1.5.8/index.php/Creating_a_project_from_scratch
* http://bulletphysics.org/mediawiki-1.5.8/index.php/Hello_World

### Tutorials
* http://bulletphysics.org/mediawiki-1.5.8/index.php/Tutorials

### Collision
* Collision Filter http://www.bulletphysics.org/mediawiki-1.5.8/index.php/Collision_Filtering
* Broadphase , 用于选择不同的碰撞检测算法 http://bulletphysics.org/mediawiki-1.5.8/index.php/Broadphase
* Collision Test, Contact Callback, Trigger, http://www.bulletphysics.org/mediawiki-1.5.8/index.php/Collision_Callbacks_and_Triggers
  eg:ConvexSweepTest, ContactTest
* CCD (Continuous Collision Detection) https://www.panda3d.org/manual/index.php?title=Bullet_Continuous_Collision_Detection&language=python and
  http://bulletphysics.org/mediawiki-1.5.8/index.php/Anti_tunneling_by_Motion_Clamping
* Raycast http://bulletphysics.org/mediawiki-1.5.8/index.php/Using_RayTest

### Step Simulation
* Stepping http://www.bulletphysics.org/mediawiki-1.5.8/index.php?title=Stepping_The_World
* Stepping Callbacks, 类似于Unity中的 FixedUpdate()，framerate independate, http://www.bulletphysics.org/mediawiki-1.5.8/index.php/Simulation_Tick_Callbacks

### Useful Code Snippets
* http://www.bulletphysics.org/mediawiki-1.5.8/index.php/Code_Snippets

看完这些基本对Bullet 物理引擎有了大致的了解

## 在项目中的使用

### 导出和加载 Mesh 文件
要在服务器中使用 Bullet，那么就要知道相应的碰撞体的形状大小信息，场景中那些初始的包围圈通过导出到 obj 文件中，在加载到服务器中就可以创建相应的 Rigidbody 了。
关于如何导出和加载 obj 文件，网上有现成的代码，可以参考http://wiki.unity3d.com/index.php/ObjExporter 和 https://github.com/chrisjansson/ObjLoader

### 创建 CollisionShap 和 RigidBody
```c++
btRigidBody BasicExample::createRigidBodyNew(float mass, btVector3 pos, bool isTrigger, bool useG, btCollisionShape* shap, bool restrictY)
{
    btTransform transform;
    transform.setIdentity();
    transform.setOrigin(pos);
    return createRigidBodyNew(mass, transform, isTrigger, useG, shap, restrictY);
}

btRigidBody BasicExample::createRigidBodyNew(float mass, const btTransform& transform, bool isTrigger, bool useG, btCollisionShape* shape, bool restrictY)
{
    bool isDynamic = mass != 0.f;
	btVector3 localInertia(0, 0, 0);
	if (isDynamic)
		shape->calculateLocalInertia(mass, localInertia);

	//using motionstate is recommended, it provides interpolation capabilities, and only synchronizes 'active' objects

#define USE_MOTIONSTATE 1
#ifdef USE_MOTIONSTATE
	btDefaultMotionState* myMotionState = new btDefaultMotionState(transform);

    btRigidBody::btRigidBodyConstructionInfo cInfo(mass, myMotionState, shape, localInertia);

	btRigidBody* body = new btRigidBody(cInfo);
	//body->setContactProcessingThreshold(m_defaultContactProcessingThreshold);

#else
	btRigidBody* body = new btRigidBody(mass, 0, shape, localInertia);
	body->setWorldTransform(startTransform);
#endif//
    
    if(isTrigger)
        body->setCollisionFlags(body->getCollisionFlags() | btCollisionObject::CF_NO_CONTACT_RESPONSE);
    
    if(restrictY)
        body->setLinearFactor(btVector3(1,0,1));

	m_dynamicsWorld->addRigidBody(body);
    
    return *body;
}
```

### 检测碰撞
``` c++
int numManifolds = world->getDispatcher()->getNumManifolds();
    for (int i = 0; i < numManifolds; i++)
    {
        btPersistentManifold* contactManifold =  
        world->getDispatcher()->getManifoldByIndexInternal(i);
        const btCollisionObject* obA = contactManifold->getBody0();
        const btCollisionObject* obB = contactManifold->getBody1();

        int numContacts = contactManifold->getNumContacts();
        for (int j = 0; j < numContacts; j++)
        {
            btManifoldPoint& pt = contactManifold->getContactPoint(j);
            if (pt.getDistance() < 0.f)
            {
                const btVector3& ptA = pt.getPositionWorldOnA();
                const btVector3& ptB = pt.getPositionWorldOnB();
                const btVector3& normalOnB = pt.m_normalWorldOnB;
            }
        }
    }
```

## 补充
碰撞测试在游戏中也非常有用，例如炸弹在爆炸的时候对周围一定范围内的 Npc 造成伤害，这时候就可以用 Bullet 中的 **contactTest** 来实现
```c++
struct ContactSensorCallback : public btCollisionWorld::ContactResultCallback {
    
    //! Constructor, pass whatever context you want to have available when processing contacts
    /*! You may also want to set m_collisionFilterGroup and m_collisionFilterMask
     *  (supplied by the superclass) for needsCollision() */
    ContactSensorCallback(btRigidBody& tgtBody , void* context /*, ... */)
    : btCollisionWorld::ContactResultCallback(), body(tgtBody), ctxt(context) { }
    
    btRigidBody& body; //!< The body the sensor is monitoring
    void* ctxt; //!< External information for contact processing
    //保存结果
    btAlignedObjectArray<btRigidBody*> result;
    
    //! If you don't want to consider collisions where the bodies are joined by a constraint, override needsCollision:
    /*! However, if you use a btCollisionObject for #body instead of a btRigidBody,
     *  then this is unnecessary—checkCollideWithOverride isn't available */
    virtual bool needsCollision(btBroadphaseProxy* proxy) const {
        // superclass will check m_collisionFilterGroup and m_collisionFilterMask
        if(!btCollisionWorld::ContactResultCallback::needsCollision(proxy))
            return false;
        // if passed filters, may also want to avoid contacts between constraints
        return body.checkCollideWithOverride(static_cast<btCollisionObject*>(proxy->m_clientObject));
    }
    
    //! Called with each contact for your own processing (e.g. test if contacts fall in within sensor parameters)
    virtual btScalar addSingleResult(btManifoldPoint& cp,
                                     const btCollisionObjectWrapper* colObj0,int partId0,int index0,
                                     const btCollisionObjectWrapper* colObj1,int partId1,int index1)
    {
        btVector3 pt; // will be set to point of collision relative to body
        btRigidBody* other;
        if(colObj0->m_collisionObject==&body) {
            pt = cp.m_localPointA;
            other = const_cast<btRigidBody*>(reinterpret_cast<const btRigidBody*>(colObj1->m_collisionObject));
        } else {
            assert(colObj1->m_collisionObject==&body && "body does not match either collision object");
            pt = cp.m_localPointB;
            other = const_cast<btRigidBody*>(reinterpret_cast<const btRigidBody*>(colObj1->m_collisionObject));
        }
        //check condition on other, if satisfied, add other to result
        result.push_back(other);
        // do stuff with the collision point
        return 0; // There was a planned purpose for the return value of addSingleResult, but it is not used so you can ignore it.
    }
};


// USAGE:
btRigidBody* tgtBody /* = ... */;
YourContext foo;
ContactSensorCallback callback(*tgtBody, foo);
world->contactTest(tgtBody,callback);
return callback.result;
```