---
title: MRC/ARC & 引用计数管理
keywords: iOS面试
date: 2019-04-28 13:47:40
categories: 

- 面试
  tags:
- 内存管理
  comments: true
---

# MRC

MRC:通过手动引用计数来进行对象内存管理。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-1.png)

- alloc:分配内存空间
- retain:引用计数+1
- release:-1操作
- retainCount:当前对象引用计数值
- autorelease:对象会在autoreleasepool结束的时候-1
- dealloc:mrc中调用的时需要调用super dealloc;

# ARC

Q:什么是ARC?

- ARC是LLVM和Runtime协作的结果。
- ARC禁止手动调用retain/release/retainCount/dealloc
- ARC新增`weak, strong`属性关键字。



# 引用计数

## 实现原理分析

- alloc
- retain
- release
- retainCount
- dealloc

这些系统实现原理。

### alloc实现

- 经过一系列调用，最终调用了c函数的calloc.
- 此时并没有设置引用计数为1。

### retain实现

```c#
Sidetable& table = SideTables()[this];
//我们根据对象指针在SideTables中经过hash计算快速得到这个对象的sidetable
size_t& refcntStorage = table.refcnts[this];
//从table的成员变量引用计数表refcnts.
refcntStorage += SIDE_TABLE_RC_ONE;
//给引用计数进行+1操作。这里加的值不是实际的1，size_t前两位没有存值，相当于进行一个偏移。
```

`Q:在进行retain操作的时候，系统是如何查找对应的引用计数的？`

A:经过两次Hash查找，找到对应的引用计数值，然后+1.

### release实现

```c#
Sidetable& table = SideTables()[this];
//我们根据对象指针在SideTables中经过hash计算快速得到这个对象的sidetable
RefcountMap::iterator it = table.refcnts.find[this];
//查找引用计数表
it->second -= SIDE_TABLE_RC_ONE;
//给引用计数进行-1操作。
```

### retainCount实现

```c#
Sidetable& table = SideTables()[this];
//我们根据对象指针在SideTables中经过hash计算快速得到这个对象的sidetable
size_t refcnt_result = 1;
RefcountMap::iterator it = table.refcnts.find[this];
//查找引用计数表
refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
//这里因为第一次alloc的时候，it是nil,但是refcnt是为1，因此使用retainCount的查看的时候为1；
```

### dealloc实现

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-2.png)

> dealloc首先调用私有函数`_objc_rootDealloc()`,这个函数会调用 `rootDealloc()`,在rootDealloc()函数中会判断当前对象是否可以直接释放：
>
> 1. nonpointer_isa：是否使用了非指针型isa
> 2. weakly_referenced:是否有weak指针指向
> 3. has_assoc:是否有关联对象
> 4. has_cxx_dtor:当前对象内部实现是否涉及C++相关内容，以及当前对象是否使用ARC管理内存。
> 5. has_sidetable_rc:当前对象的引用计数是通过sidetable中的引用计数表。
>
> 当上面五个都不存在的情况下，可以调用c函数直接释放，否则调用 `object_dispose()`清除。

`为什么要做上面的五个判断？`

#### Object_dispose()实现

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-3.png)

#### Objc_destructInstance()实现逻辑

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-4.png)

> 首先判断：
>
> 1. hasCxxDtor
> 2. hasAssociatedObjects

`Q：我们通过关联对象为一个类添加了实例对象，在dealloc方法中是否需要移除关联对象？`

A:系统dealloc中会自动判断是否有关联对象，有的话，会自动移除。不需要手动移除。

#### clearDeallocating()实现

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-5.png)

`Q：如果一个对象有weak指针指向它，当这个对象dealloc或者废弃之后，它的weak指针为什么自动置为nil?`

A：在dealloc中有这样的操作。

# 弱引用管理

## 一个weak变量是如何添加到弱引用表中的？

Q:被声明为__weak的对象指针，经过编译器编译之后，会调用objc_initWeak()方法，经过一系列函数调用栈，会调用weak_register_no_lock进行弱引用添加，具体添加的位置，是通过hash算法进行位置查找，如果说查找的对应位置中有这个弱引用数组，把新的弱引用变量添加到数组中，如果没有，我们重新创建一个弱引用数组，把第0个位置添加我们新的弱引用变量，其余的位置设置为nil.

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-6.png)

**<u>添加weak变量过程</u>**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-7.png)



我们源码分析：`weak_register_no_lock()`:

```c++
/** 
 * Initialize a fresh weak pointer to some object location. 
 * It would be used for code like: 
 *
 * (The nil case) 
 * __weak id weakPtr;
 * (The non-nil case) 
 * NSObject *o = ...;
 * __weak id weakPtr = o;
 * 
 * This function IS NOT thread-safe with respect to concurrent 
 * modifications to the weak variable. (Concurrent weak clear is safe.)
 *
 * @param location Address of __weak ptr. 
 * @param newObj Object ptr. 
 */
id
objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<false/*old*/, true/*new*/, true/*crash*/>
        (location, (objc_object*)newObj);
}
```

```c++
// Update a weak variable.
// If HaveOld is true, the variable has an existing value 
//   that needs to be cleaned up. This value might be nil.
// If HaveNew is true, there is a new value that needs to be 
//   assigned into the variable. This value might be nil.
// If CrashIfDeallocating is true, the process is halted if newObj is 
//   deallocating or newObj's class does not support weak references. 
//   If CrashIfDeallocating is false, nil is stored instead.
template <bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    assert(HaveOld  ||  HaveNew);
    if (!HaveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    if (HaveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (HaveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);

    if (HaveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    if (HaveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any.
    if (HaveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (HaveNew) {
        newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table, 
                                                      (id)newObj, location, 
                                                      CrashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

    return (id)newObj;
}

```

```c++
/** 
 * Registers a new (object, weak pointer) pair. Creates a new weak
 * object entry if it does not exist.
 * 
 * @param weak_table The global weak table.
 * @param referent The object pointed to by the weak reference.
 * @param referrer The weak pointer address.
 */
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry;
        new_entry.referent = referent;
        new_entry.out_of_line = 0;
        new_entry.inline_referrers[0] = referrer;
        for (size_t i = 1; i < WEAK_INLINE_COUNT; i++) {
            new_entry.inline_referrers[i] = nil;
        }
        
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
```

```c++
/** 
 * Return the weak reference table entry for the given referent. 
 * If there is no entry for referent, return NULL. 
 * Performs a lookup.
 *
 * @param weak_table 
 * @param referent The object. Must not be nil.
 * 
 * @return The table of weak referrers to this object. 
 */
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    assert(referent);

    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t index = hash_pointer(referent) & weak_table->mask;
    size_t hash_displacement = 0;
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
}
```

## 系统是如何清除weak变量，同时设置指向为nil?

Q：当一个对象被dealloc后，在dealloc函数中会调用weak_clear_no_lock()函数，在函数内部根据对象指针查找弱引用表，把当前对象的所有弱引用对象拿出来是一个数组，遍历这个数组，全部置为nil.

**<u>清除weak变量过程：</u>**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/6-2-8.png)

```c++
/** 
 * Called by dealloc; nils out all weak pointers that point to the 
 * provided object so that they can no longer be used.
 * 
 * @param weak_table 
 * @param referent The object being deallocated. 
 */
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
      //referrers是取到的所有的弱引用变量
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    weak_entry_remove(weak_table, entry);
}
```

