# pthread_mutex_lock

Mutex Lock 的数据结构

```C
static inline pthread_mutex_internal_t* __get_internal_mutex(pthread_mutex_t* mutex_interface) {
  return reinterpret_cast<pthread_mutex_internal_t*>(mutex_interface);
}

struct pthread_mutex_internal_t {
    _Atomic(uint16_t) state;
    uint16_t __pad;
    union {
        atomic_int owner_tid;
        PIMutex pi_mutex;
    };
    char __reserved[28];

    PIMutex& ToPIMutex() {
        return pi_mutex;
    }

    void FreePIMutex() {
    }
} __attribute__((aligned(4)));

// Priority Inheritance mutex implementation
struct PIMutex {
  // mutex type, can be 0 (normal), 1 (recursive), 2 (errorcheck), constant during lifetime
  uint8_t type;
  // process-shared flag, constant during lifetime
  bool shared;
  // <number of times a thread holding a recursive PI mutex> - 1
  uint16_t counter;
  // owner_tid is read/written by both userspace code and kernel code. It includes three fields:
  // FUTEX_WAITERS, FUTEX_OWNER_DIED and FUTEX_TID_MASK.
  atomic_int owner_tid;
};
```

其中 state 是一个十六位的无符号类型。


```
+-------------------------------------------+
| PI  |shared|                        |state|
+-----+------+------------------------+-----+
|15|14|  13  |12|11|10|9|8|7|6|5|4|3|2| 1|0 |
```

PI:

- 00: 非 PI 类型
- 01: 检错锁（不可重入锁）
- 11: PI 类型


State:
- 0: 未加锁
- 1: 已加锁
- 2: 加锁，且有休眠的线程在等待锁。


```C

```