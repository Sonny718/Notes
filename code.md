## 随笔


- [随笔](#随笔)
- [对象池 Object pool](#对象池-object-pool)
- [cyber/base/macros.h](#cyberbasemacrosh)
- [cyber/base/atomic_fifo.h](#cyberbaseatomic_fifoh)
<!-- 2. [Prerequisites](#prerequisites)
    - [Basic Requirements](#basic-requirements)
    - [Individual Version Requirements](#individual-version-requirements)
1. [Architecture](#architecture)
2. [Installation](#installation)
3. [Documents](#documents) -->


## 对象池 Object pool
[深入浅出对象池(Object Pool)](https://my.oschina.net/bobwei/blog/646192)

Wheel 实现方式
```C++
// Copyright 2021 daohu527@gmail.com
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

//  Created Date: 2021-8-20
//  Author: daohu527

#pragma once

#include <cassert>
#include <functional>
#include <list>
#include <memory>
#include <mutex>
#include <utility>


namespace wheel {
namespace base {

template <class C, class P = C*>
class ObjectFactory {
 public:
  template <class ...Args>
  P createObject(Args&&... args) {
    return new C(std::forward<Args>(args)...);
  }

  void destroyObject(P ptr) {
    delete ptr;
  }
};

template <class C>
class ObjectFactory <C, std::shared_ptr<C>> {
 public:
  template <class ...Args>
  std::shared_ptr<C> createObject(Args&&... args) {
    return std::make_shared<C>(std::forward<Args>(args)...);
  }

  void destroyObject(std::shared_ptr<C> ptr) {}
};

template <class C, class P = C*, class F = ObjectFactory<C, P>>
class ObjectPool {
 public:
  using size_type = std::size_t;
  using Predicate = std::function<bool(P)>;

  ObjectPool(const ObjectPool&) = delete;
  ObjectPool& operator=(const ObjectPool&) = delete;

  ObjectPool(size_type capacity, size_type peak_capacity)
      : capacity_(capacity),
        peak_capacity_(peak_capacity),
        size_(0) {
    assert(capacity_ <= peak_capacity_);
  }

  ObjectPool(size_type capacity, size_type peak_capacity, F factory)
      : capacity_(capacity),
        peak_capacity_(peak_capacity),
        size_(0),
        factory_(factory) {
    assert(capacity_ <= peak_capacity_);
  }

  ObjectPool(size_type capacity,
             size_type peak_capacity,
             F factory,
             Predicate pred)
      : capacity_(capacity),
        peak_capacity_(peak_capacity),
        size_(0),
        factory_(factory),
        pred_(pred) {
    assert(capacity_ <= peak_capacity_);
  }

  ~ObjectPool() {
    typename std::list<P>::iterator it = pool_.begin();
    while (it != pool_.end()) {
      factory_.destroyObject(*it);
      ++it;
    }
  }

  P borrowObject();

  void returnObject(P ptr);

  size_type capacity() const {
    return capacity_;
  }

  size_type peak_capacity() const {
    return peak_capacity_;
  }

  size_type size() const {
    std::lock_guard<std::mutex> lock(mutex_);
    return size_;
  }

  size_type available() const {
    std::lock_guard<std::mutex> lock(mutex_);
    return peak_capacity_ - size_ + pool_.size();
  }

 private:
  const size_type capacity_;
  const size_type peak_capacity_;
  size_type size_;
  std::list<P> pool_;

  F factory_;
  Predicate pred_;

  mutable std::mutex mutex_;
};

template <class C, class P, class F>
P ObjectPool<C, P, F>::borrowObject() {
  std::lock_guard<std::mutex> lock(mutex_);

  if (!pool_.empty()) {
    P ptr = pool_.front();
    pool_.pop_front();
    return ptr;
  } else if (size_ < peak_capacity_) {
    P ptr = factory_.createObject();
    size_++;
    return ptr;
  } else {
    return nullptr;
  }
}

template <class C, class P, class F>
void ObjectPool<C, P, F>::returnObject(P ptr) {
  std::lock_guard<std::mutex> lock(mutex_);

  if (size_ < capacity_) {
    pool_.push_front(ptr);
  } else {
    factory_.destroyObject(ptr);
    size_--;
  }
}

}  // namespace base
}  // namespace wheel

```

其中使用了[可变参数模板](https://blog.csdn.net/qq_44194231/article/details/113436004)，传入不同类型的参数，递归输出。

Apollo实现方式

Apollo CyberRT架构
TODO：架构图


代码走读
[Cyber RT 3.5走读](https://blog.csdn.net/weixin_44450715/article/details/86099504)
以下引用均为Cyber RT 3.5版本
## cyber/base/macros.h
```C++
/******************************************************************************
 * Copyright 2018 The Apollo Authors. All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *****************************************************************************/

#ifndef CYBER_BASE_MACROS_H_
#define CYBER_BASE_MACROS_H_

#include <cstdlib>
#include <new>

//
// Study: __builtin_expect is a function to let the compiler know the chance of the condition to be true
// , the compiler can do optimization in assembly level using this information
//__GNUC__ GCC版本
#if __GNUC__ >= 3
#define likely(x) (__builtin_expect((x), 1))
#define unlikely(x) (__builtin_expect((x), 0))
#else
#define likely(x) (x)
#define unlikely(x) (x)
#endif
```
[__builtin_expect](https://blog.csdn.net/shuimuniao/article/details/8017971?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link)
作用：优化程序，暂时不深入

```C++

// Study: The size of cache line, cache line is the a block that will be fetch from memory to cache
#define CACHELINE_SIZE 64

// Study: Meta-programming, used SFINTE. When the T type have the `func` function, it will be true,
//        Otherwise it is false (since size of int is not 1)
//        basically used to determine the exist of the func for T in compile time
//        Can be mixed with the stl traits
#define DEFINE_TYPE_TRAIT(name, func)                            \
  template <typename T>                                          \
  class name {                                                   \
   private:                                                      \
    template <typename Class>                                    \
    static char Test(decltype(&Class::func)*);                   \
    template <typename>                                          \
    static int Test(...);                                        \
                                                                 \
   public:                                                       \
    static constexpr bool value = sizeof(Test<T>(nullptr)) == 1; \
  };                                                             \
                                                                 \
  template <typename T>                                          \
  constexpr bool name<T>::value;


```
TODO：
了解缓存Cache和缓存行Cache line
[缓存](https://blog.csdn.net/qq_21125183/article/details/80590934) 
元编程
[元编程](https://www.zhihu.com/question/23856985)
等使用时深入了解

```C++

// Study: Call the processer to pause (no operation)
//        The different of rep;nop; to nop;nop; is that processor can optimize with this
inline void cpu_relax() { asm volatile("rep; nop" ::: "memory"); }

// Study: Allocate memory
inline void* CheckedMalloc(size_t size) {
  void* ptr = std::malloc(size);
  if (!ptr) {
    throw std::bad_alloc();
  }
  return ptr;
}

// Study: Allocate memory and Clean location
inline void* CheckedCalloc(size_t num, size_t size) {
  void* ptr = std::calloc(num, size);
  if (!ptr) {
    throw std::bad_alloc();
  }
  return ptr;
}

#endif  // CYBER_BASE_MACROS_H_

```
分配内存
Q:为什么用内联?

## cyber/base/atomic_fifo.h

```C++
/******************************************************************************
 * Copyright 2018 The Apollo Authors. All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *****************************************************************************/

#ifndef CYBER_BASE_ATOMIC_FIFO_H_
#define CYBER_BASE_ATOMIC_FIFO_H_

#include <atomic>
#include <cstdlib>
#include <cstring>
#include <iostream>

#include "cyber/base/macros.h"

namespace apollo {
namespace cyber {

template <typename T>
class AtomicFIFO {
 private:
  struct Node {
    T value;
  };

 public:
  // Study: Singleton
  static AtomicFIFO *GetInstance(int cap = 100) {
    static AtomicFIFO *inst = new AtomicFIFO(cap);
    return inst;
  }

  bool Push(const T &value);
  bool Pop(T *value);
  // insert();

 private:
  Node *node_arena_;
  // Study: Align to maximize the memory access speed
  alignas(CACHELINE_SIZE) std::atomic<uint32_t> head_;
  alignas(CACHELINE_SIZE) std::atomic<uint32_t> commit_;
  alignas(CACHELINE_SIZE) std::atomic<uint32_t> tail_;
  int capacity_;

  // Study: Only allow Singleton
  explicit AtomicFIFO(int cap);
  ~AtomicFIFO();
  AtomicFIFO(AtomicFIFO &) = delete;
  AtomicFIFO &operator=(AtomicFIFO &) = delete;
};

```
Q:为什么使用单例模式,而且使用public
  AtomicFIFO(AtomicFIFO &) = delete;
  AtomicFIFO &operator=(AtomicFIFO &) = delete;
  
TODO:
了解 alignas
explicit 关键字的作用就是防止类构造函数的隐式自动转换.
[C++ explicit关键字详解](https://blog.csdn.net/guoyunfei123/article/details/89003369)

```C++
template <typename T>
AtomicFIFO<T>::AtomicFIFO(int cap) : capacity_(cap) {
  node_arena_ = static_cast<Node *>(malloc(capacity_ * sizeof(Node)));
  memset(node_arena_, 0, capacity_ * sizeof(Node));

  // Study: Set value to 0
  head_.store(0, std::memory_order_relaxed);
  tail_.store(0, std::memory_order_relaxed);
  commit_.store(0, std::memory_order_relaxed);
}

template <typename T>
AtomicFIFO<T>::~AtomicFIFO() {
  if (node_arena_ != nullptr) {
    for (int i = 0; i < capacity_; i++) {
      // Study: Call the T destructor manaully, since it is puted in the malloc region, it will not auto destruct
      node_arena_[i].value.~T();
    }
    free(node_arena_);
  }
}

template <typename T>
bool AtomicFIFO<T>::Push(const T &value) {
  uint32_t oldt, newt;

  // Study:  Try push until success, return false if queue full
  oldt = tail_.load(std::memory_order_acquire);
  do {
    uint32_t h = head_.load(std::memory_order_acquire);
    uint32_t t = tail_.load(std::memory_order_acquire);

    if (((t + 1) % capacity_) == h) return false;

    newt = (oldt + 1) % capacity_;
    // Study:  If success, set tail_ to newt, otherwise set oldt to current tail_
    //         Ensure tails value sync
  } while (!tail_.compare_exchange_weak(oldt, newt, std::memory_order_acq_rel,
                                        std::memory_order_acquire));

  (node_arena_ + oldt)->value = value;

  // Study:  commit_ is basically same as tail_, but it is used in pop.
  //         It can let the pop operation not block the push core part
  while (unlikely(commit_.load(std::memory_order_acquire) != oldt)) cpu_relax();

  // Study:  After commit, this value can be pop in Pop()
  commit_.store(newt, std::memory_order_release);

  return true;
}

template <typename T>
bool AtomicFIFO<T>::Pop(T *value) {
  uint32_t oldh, newh;

  oldh = head_.load(std::memory_order_acquire);

  // Study:  Basically same logic as the push part, try pop until success. Return false if no element
  do {
    uint32_t h = head_.load(std::memory_order_acquire);
    uint32_t c = commit_.load(std::memory_order_acquire);

    if (h == c) return false;

    newh = (oldh + 1) % capacity_;

    *value = (node_arena_ + oldh)->value;
  } while (!head_.compare_exchange_weak(oldh, newh, std::memory_order_acq_rel,
                                        std::memory_order_acquire));

  return true;
}

}  // namespace cyber
}  // namespace apollo

#endif  // CYBER_BASE_ATOMIC_FIFO_H_


```


