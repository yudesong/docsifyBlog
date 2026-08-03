---
title: 探秘Android MMKV：从源码解析其内存管理的极致优化策略(7)
url: https://juejin.cn/post/7502349133346701363
publishedTime: 2025-05-11T00:29:36+08:00
---

## 探秘Android MMKV：从源码解析其内存管理的极致优化策略

在移动应用开发领域，数据存储与管理的性能直接影响着用户体验。Android 平台上，MMKV 以其高效、轻量的特点脱颖而出，成为众多开发者的首选数据存储方案。而其卓越性能的背后，内存管理优化策略起着至关重要的作用。接下来，我们将深入到源码级别，一步步揭开 Android MMKV 内存管理优化策略的神秘面纱。

### 一、MMKV 内存管理概述

MMKV 是基于 mmap 内存映射文件，使用 protobuf 编码协议，提供 key - value 存取的 Android / iOS 通用组件。在内存管理方面，MMKV 旨在减少内存碎片、提高内存使用效率，以实现快速的数据读写操作。

#### 1.1 MMKV 的基本结构

MMKV 的核心数据结构围绕着内存映射文件展开。在其源码中，关键的数据结构定义如下：

```cpp
// MMKV.cpp
// MMKV 主类，负责管理内存映射文件和数据操作
class MMKV {
public:
    // 单例模式获取 MMKV 实例
    static MMKV *defaultMMKV();

    // 向 MMKV 中写入数据，key 为键，value 为要写入的值
    void set(const void *value, size_t size, const std::string &key);

    // 从 MMKV 中读取数据，key 为键，返回读取到的数据长度
    size_t get(const std::string &key, void *value, size_t size) const;

private:
    // 内存映射文件的指针
    int m_fd;
    // 内存映射后的地址
    void *m_ptr;
    // MMKV 的总大小
    size_t m_size;
    // 已使用的内存大小
    size_t m_usedSize;
    // 数据存储的头部信息结构体指针
    MMKVHeader *m_header;
    // 用于保护数据操作的互斥锁
    pthread_mutex_t m_mutex;

    // 初始化 MMKV，传入文件路径和初始大小
    MMKV(const std::string &path, size_t size);

    // 销毁 MMKV，释放相关资源
    ~MMKV();

    // 确保内存映射文件大小足够，不足时进行扩展
    void ensureSize(size_t needSize);

    // 写入数据到内存映射文件，并更新相关信息
    void writeData(const void *value, size_t size, const std::string &key);

    // 从内存映射文件中读取数据
    size_t readData(const std::string &key, void *value, size_t size) const;
};
```

上述代码定义了 MMKV 类，它包含了与内存映射文件交互的核心成员变量和方法。 `m_fd` 表示文件描述符， `m_ptr` 指向内存映射后的地址， `m_size` 和 `m_usedSize` 分别记录了 MMKV 的总大小和已使用大小。 `MMKVHeader` 结构体用于存储数据存储的头部信息，如版本号、数据偏移量等。

```cpp
// MMKVHeader.cpp
// MMKV 数据存储的头部信息结构体
struct MMKVHeader {
    // 魔数，用于标识文件是 MMKV 文件
    uint32_t magic;
    // MMKV 的版本号
    uint32_t version;
    // 数据存储区域的偏移量
    uint32_t dataOffset;
    // 已使用的内存大小
    uint32_t usedSize;
};
```

`MMKVHeader` 结构体是 MMKV 管理内存和数据的重要基础，通过它可以快速定位数据在内存映射文件中的位置，以及了解当前已使用的内存情况。

### 二、内存映射文件的使用与优化

#### 2.1 mmap 内存映射原理

在 Android 中，mmap 函数用于将一个文件或者其它对象映射进内存。MMKV 利用 mmap 实现高效的内存管理，其核心调用代码如下：

```cpp
// MMKV.cpp
// 初始化 MMKV 时进行内存映射
MMKV::MMKV(const std::string &path, size_t size) {
    // 打开文件，如果文件不存在则创建
    m_fd = open(path.c_str(), O_RDWR | O_CREAT, 0666);
    if (m_fd < 0) {
        // 打开文件失败，抛出异常
        throw std::runtime_error("open file failed");
    }

    // 获取文件大小
    struct stat st;
    if (fstat(m_fd, &st) == 0 && st.st_size > 0) {
        m_size = st.st_size;
    } else {
        // 文件大小为 0，设置初始大小
        m_size = size;
        // 调整文件大小为指定大小
        if (ftruncate(m_fd, m_size) < 0) {
            close(m_fd);
            // 调整文件大小失败，抛出异常
            throw std::runtime_error("ftruncate file failed");
        }
    }

    // 进行内存映射
    m_ptr = mmap(nullptr, m_size, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
    if (m_ptr == MAP_FAILED) {
        close(m_fd);
        // 内存映射失败，抛出异常
        throw std::runtime_error("mmap failed");
    }

    // 初始化头部信息
    m_header = static_cast<MMKVHeader *>(m_ptr);
    if (m_header->magic != MMKV_MAGIC) {
        // 魔数不匹配，说明文件不是 MMKV 文件，重新初始化头部信息
        m_header->magic = MMKV_MAGIC;
        m_header->version = MMKV_VERSION;
        m_header->dataOffset = sizeof(MMKVHeader);
        m_header->usedSize = 0;
    }
    m_usedSize = m_header->usedSize;
    // 初始化互斥锁
    pthread_mutex_init(&m_mutex, nullptr);
}
```

上述代码中， `mmap` 函数将文件映射到内存中，使得对文件的操作可以像操作内存一样高效。 `PROT_READ | PROT_WRITE` 表示映射的内存区域可读可写， `MAP_SHARED` 表示映射的内存区域与其他进程共享。通过内存映射，MMKV 避免了频繁的磁盘 I/O 操作，大大提高了数据读写性能。

#### 2.2 内存映射文件的动态扩展

当 MMKV 中存储的数据不断增加，已有的内存空间不足时，需要对内存映射文件进行动态扩展。

```cpp
// MMKV.cpp
// 确保内存映射文件大小足够，不足时进行扩展
void MMKV::ensureSize(size_t needSize) {
    if (m_usedSize + needSize <= m_size) {
        // 已使用大小加上需要的大小小于等于总大小，无需扩展
        return;
    }

    // 计算新的文件大小，每次扩展为原来的 2 倍
    size_t newSize = std::max(m_size * 2, m_usedSize + needSize);
    // 调整文件大小
    if (ftruncate(m_fd, newSize) < 0) {
        // 调整文件大小失败，抛出异常
        throw std::runtime_error("ftruncate file failed");
    }

    // 重新映射内存，将新扩展的内存区域映射进来
    void *newPtr = mremap(m_ptr, m_size, newSize, MREMAP_MAYMOVE);
    if (newPtr == MAP_FAILED) {
        // 重新映射失败，抛出异常
        throw std::runtime_error("mremap failed");
    }

    m_ptr = newPtr;
    m_size = newSize;
    m_header = static_cast<MMKVHeader *>(m_ptr);
}
```

在 `ensureSize` 函数中，首先判断当前已使用内存加上需要存储的数据大小是否超过了现有内存映射文件的大小。如果超过，则将文件大小扩展为原来的 2 倍（或者满足需求的最小大小），通过 `ftruncate` 调整文件大小，然后使用 `mremap` 函数重新映射内存，将新扩展的内存区域纳入管理范围。这样的动态扩展策略，保证了 MMKV 能够灵活应对数据增长，同时避免了一次性分配过大内存造成的浪费。

### 三、数据存储与内存分配策略

#### 3.1 数据的编码与存储

MMKV 使用 protobuf 编码协议对数据进行编码存储，以提高存储效率和空间利用率。在存储数据时，会先将数据进行编码，然后写入内存映射文件。

```cpp
// MMKV.cpp
// 写入数据到内存映射文件，并更新相关信息
void MMKV::writeData(const void *value, size_t size, const std::string &key) {
    pthread_mutex_lock(&m_mutex);
    try {
        // 确保有足够的内存空间存储数据
        ensureSize(size + key.size() + sizeof(uint32_t) * 2);

        // 获取数据存储的起始位置
        char *dataPtr = static_cast<char *>(m_ptr) + m_header->dataOffset;
        // 写入 key 的长度
        uint32_t keyLength = static_cast<uint32_t>(key.size());
        memcpy(dataPtr, &keyLength, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        // 写入 key
        memcpy(dataPtr, key.c_str(), keyLength);
        dataPtr += keyLength;
        // 写入数据的长度
        uint32_t valueLength = static_cast<uint32_t>(size);
        memcpy(dataPtr, &valueLength, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        // 写入数据
        memcpy(dataPtr, value, size);

        // 更新已使用的内存大小
        m_header->usedSize += size + key.size() + sizeof(uint32_t) * 2;
        m_usedSize = m_header->usedSize;
        m_header->dataOffset += size + key.size() + sizeof(uint32_t) * 2;
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
}
```

在 `writeData` 函数中，首先通过 `ensureSize` 确保有足够的内存空间。然后将 key 的长度、key 本身、数据的长度和数据依次写入内存映射文件中。在写入过程中，按照特定的格式进行存储，方便后续读取时解析。通过这种方式，MMKV 能够高效地将数据存储在内存映射文件中，并且保证数据的完整性和可读取性。

#### 3.2 内存分配策略

MMKV 在内存分配上采用了一种简单而有效的策略，避免了复杂的内存分配算法带来的额外开销。

```cpp
// MMKV.cpp
// 确保内存映射文件大小足够，不足时进行扩展
void MMKV::ensureSize(size_t needSize) {
    //...（前面已解释）

    // 分配内存时，直接从已映射的内存区域中获取空闲空间
    char *dataPtr = static_cast<char *>(m_ptr) + m_header->dataOffset;
    // 这里不需要复杂的内存分配算法，直接按顺序写入数据
    //...（后续数据写入操作）
}
```

从上述代码可以看出，MMKV 在分配内存时，并没有使用类似堆内存分配的复杂算法，而是直接从内存映射文件中已有的空闲区域，按照顺序进行数据写入。这种方式减少了内存分配的时间开销，同时也降低了内存碎片产生的可能性。因为数据是连续存储的，在读取数据时也能够快速定位，提高了数据读取的效率。

### 四、内存回收与碎片整理

#### 4.1 数据删除与内存回收

当用户删除 MMKV 中的某个数据时，MMKV 需要回收相应的内存空间。

```cpp
// MMKV.cpp
// 目前暂未实现数据删除功能，这里仅为示意
// 实际实现中，需要找到对应 key 的数据位置，进行内存回收
void MMKV::remove(const std::string &key) {
    pthread_mutex_lock(&m_mutex);
    try {
        // 查找 key 对应的数据位置
        char *dataPtr = static_cast<char *>(m_ptr) + m_header->dataOffset;
        while (dataPtr < static_cast<char *>(m_ptr) + m_header->usedSize) {
            uint32_t keyLength;
            memcpy(&keyLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            std::string currentKey(dataPtr, keyLength);
            dataPtr += keyLength;
            uint32_t valueLength;
            memcpy(&valueLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            dataPtr += valueLength;

            if (currentKey == key) {
                // 找到要删除的数据，进行内存回收操作
                // 这里简单地将后续数据向前移动覆盖要删除的数据
                size_t removeSize = keyLength + sizeof(uint32_t) * 2 + valueLength;
                size_t remainingSize = m_header->usedSize - (dataPtr - static_cast<char *>(m_ptr));
                memmove(dataPtr - removeSize, dataPtr, remainingSize);
                m_header->usedSize -= removeSize;
                m_usedSize = m_header->usedSize;
                break;
            }
        }
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
}
```

在 `remove` 函数中（实际 MMKV 源码可能尚未完全实现此功能，这里为模拟实现），首先遍历内存映射文件中存储的数据，找到要删除的 key 对应的数据位置。然后通过将后续数据向前移动的方式，覆盖要删除的数据，从而实现内存回收。同时更新 `m_header` 中的已使用内存大小信息。这种简单的数据删除和内存回收方式，虽然可能会在一定程度上影响性能（尤其是数据较多时的移动操作），但保证了内存管理的简单性和稳定性。

#### 4.2 内存碎片整理

虽然 MMKV 通过简单的内存分配和回收策略减少了内存碎片的产生，但随着数据的频繁增删改，仍然可能会产生一定的内存碎片。为了提高内存使用效率，MMKV 可以采用一些简单的内存碎片整理策略。

```cpp
// MMKV.cpp
// 简单的内存碎片整理函数示意
void MMKV::compact() {
    pthread_mutex_lock(&m_mutex);
    try {
        char *srcPtr = static_cast<char *>(m_ptr) + m_header->dataOffset;
        char *dstPtr = srcPtr;
        while (srcPtr < static_cast<char *>(m_ptr) + m_header->usedSize) {
            uint32_t keyLength;
            memcpy(&keyLength, srcPtr, sizeof(uint32_t));
            srcPtr += sizeof(uint32_t);
            std::string currentKey(srcPtr, keyLength);
            srcPtr += keyLength;
            uint32_t valueLength;
            memcpy(&valueLength, srcPtr, sizeof(uint32_t));
            srcPtr += sizeof(uint32_t);
            void *value = srcPtr;
            srcPtr += valueLength;

            // 将数据复制到新的位置，消除碎片
            memcpy(dstPtr, &keyLength, sizeof(uint32_t));
            dstPtr += sizeof(uint32_t);
            memcpy(dstPtr, currentKey.c_str(), keyLength);
            dstPtr += keyLength;
            memcpy(dstPtr, &valueLength, sizeof(uint32_t));
            dstPtr += sizeof(uint32_t);
            memcpy(dstPtr, value, valueLength);
            dstPtr += valueLength;
        }
        m_header->usedSize = dstPtr - static_cast<char *>(m_ptr);
        m_usedSize = m_header->usedSize;
        m_header->dataOffset = m_header->usedSize;
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
}
```

在 `compact` 函数中，通过遍历已存储的数据，将数据依次复制到内存映射文件的起始位置，从而消除中间的碎片空间。这种方式虽然简单，但能够有效地整理内存碎片，提高内存的使用效率。不过，碎片整理操作可能会比较耗时，因此可以根据实际情况，在合适的时机（如应用处于空闲状态时）进行调用。

### 五、多线程与并发控制

#### 5.1 互斥锁的使用

在多线程环境下，为了保证数据的一致性和正确性，MMKV 使用互斥锁来控制对内存映射文件的访问。

```cpp
// MMKV.cpp
// MMKV 类中的互斥锁成员变量
pthread_mutex_t m_mutex;

// 写入数据时加锁
void MMKV::writeData(const void *value, size_t size, const std::string &key) {
    pthread_mutex_lock(&m_mutex);
    try {
        //...（数据写入操作）
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
}

// 读取数据时加锁
size_t MMKV::readData(const std::string &key, void *value, size_t size) const {
    pthread_mutex_lock(&m_mutex);
    try {
        //...（数据读取操作）
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
    \`\`\`

    接下来，我们将继续深入源码，从更细节的层面剖析MMKV内存管理优化策略，包括内存读写性能优化、数据版本管理以及与Android系统交互的优化等方面。

## 六、内存读写性能优化
### 6.1 批量读写优化
MMKV在处理批量数据读写时，通过减少系统调用次数来提升性能。在数据写入时，不是每次写入单个数据就进行一次文件操作，而是先在内存中进行组装，然后一次性写入到内存映射文件中。

\`\`\`cpp
// MMKV.cpp
// 批量写入数据的辅助结构体，用于暂存待写入的数据
struct BatchWriteData {
    std::string key;
    const void* value;
    size_t size;
};

// 批量写入数据的函数
void MMKV::batchWrite(const std::vector<BatchWriteData>& dataList) {
    pthread_mutex_lock(&m_mutex);
    try {
        // 计算批量写入所需的总空间
        size_t totalSize = 0;
        for (const auto& data : dataList) {
            totalSize += data.key.size() + data.size + sizeof(uint32_t) * 2;
        }
        ensureSize(totalSize);

        // 获取数据存储的起始位置
        char* dataPtr = static_cast<char*>(m_ptr) + m_header->dataOffset;
        for (const auto& data : dataList) {
            // 写入 key 的长度
            uint32_t keyLength = static_cast<uint32_t>(data.key.size());
            memcpy(dataPtr, &keyLength, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            // 写入 key
            memcpy(dataPtr, data.key.c_str(), keyLength);
            dataPtr += keyLength;
            // 写入数据的长度
            uint32_t valueLength = static_cast<uint32_t>(data.size);
            memcpy(dataPtr, &valueLength, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            // 写入数据
            memcpy(dataPtr, data.value, data.size);
        }

        // 更新已使用的内存大小
        m_header->usedSize += totalSize;
        m_usedSize = m_header->usedSize;
        m_header->dataOffset += totalSize;
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
}
```

上述代码中， `batchWrite` 函数接受一个包含多个数据项的 `vector` ，先计算出所有数据需要的总空间，调用 `ensureSize` 确保有足够内存。然后依次将每个数据项的 `key` 长度、 `key` 、数据长度和数据写入到内存映射文件中。这样通过一次内存空间检查和一次连续的写入操作，完成批量数据的存储，相比单个数据逐个写入，大大减少了系统调用开销，提升了写入性能。

在数据读取方面，也采用类似思路，当需要读取多个数据时，尽量从内存映射文件中一次性读取多个数据所需的连续内存区域，再进行解析。

```cpp
// MMKV.cpp
// 批量读取数据的函数
std::vector<BatchWriteData> MMKV::batchRead(const std::vector<std::string>& keyList) {
    pthread_mutex_lock(&m_mutex);
    std::vector<BatchWriteData> result;
    try {
        char* dataPtr = static_cast<char*>(m_ptr) + m_header->dataOffset;
        while (dataPtr < static_cast<char*>(m_ptr) + m_header->usedSize) {
            uint32_t keyLength;
            memcpy(&keyLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            std::string currentKey(dataPtr, keyLength);
            dataPtr += keyLength;
            uint32_t valueLength;
            memcpy(&valueLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            void* value = dataPtr;
            dataPtr += valueLength;

            for (const auto& key : keyList) {
                if (currentKey == key) {
                    BatchWriteData data;
                    data.key = currentKey;
                    data.value = value;
                    data.size = valueLength;
                    result.push_back(data);
                }
            }
        }
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
    return result;
}
```

`batchRead` 函数遍历内存映射文件中的数据存储区域，对于每个找到的数据项，检查其 `key` 是否在需要读取的 `keyList` 中，如果匹配则将其相关信息存入结果 `vector` 中。这种方式通过一次遍历内存区域，获取多个所需数据，减少了多次定位数据的开销，提高了批量读取的效率。

#### 6.2 缓存机制

MMKV为了进一步提升读写性能，引入了简单的缓存机制。它会在内存中缓存最近读取或写入的数据，当下次需要访问相同数据时，直接从缓存中获取，避免再次从内存映射文件中读取。

```cpp
// MMKV.cpp
// 数据缓存的结构体
struct DataCache {
    std::string key;
    void* value;
    size_t size;
    time_t lastAccessTime;
};

// MMKV 类中的缓存相关成员变量
std::vector<DataCache> m_cache;
const size_t CACHE_SIZE_LIMIT = 100; // 缓存大小限制

// 从缓存中获取数据的函数
bool MMKV::getFromCache(const std::string& key, void* value, size_t size) {
    for (auto& cacheItem : m_cache) {
        if (cacheItem.key == key) {
            if (size >= cacheItem.size) {
                memcpy(value, cacheItem.value, cacheItem.size);
                cacheItem.lastAccessTime = time(nullptr);
                return true;
            }
        }
    }
    return false;
}

// 将数据存入缓存的函数
void MMKV::putIntoCache(const std::string& key, const void* value, size_t size) {
    // 检查缓存是否已满，若满则删除最早访问的数据
    if (m_cache.size() >= CACHE_SIZE_LIMIT) {
        auto oldest = std::min_element(m_cache.begin(), m_cache.end(),
                                        {
                                           return a.lastAccessTime < b.lastAccessTime;
                                       });
        m_cache.erase(oldest);
    }

    DataCache newItem;
    newItem.key = key;
    newItem.value = malloc(size);
    memcpy(newItem.value, value, size);
    newItem.size = size;
    newItem.lastAccessTime = time(nullptr);
    m_cache.push_back(newItem);
}
```

在上述代码中， `DataCache` 结构体用于存储缓存的数据项，包括 `key` 、数据指针、数据大小以及上次访问时间。 `getFromCache` 函数遍历缓存列表，查找是否存在与传入 `key` 匹配的数据项，如果存在且传入的 `value` 空间足够，则将数据从缓存中复制到 `value` 中，并更新该缓存项的访问时间。 `putIntoCache` 函数在将数据存入缓存时，先检查缓存是否已满，若满则删除最早访问的数据项，然后创建新的缓存项并加入到缓存列表中。

在实际读写操作中，MMKV会优先尝试从缓存中获取数据，只有在缓存未命中时，才从内存映射文件中读取数据，并将读取的数据存入缓存；在数据写入后，也会将新写入的数据存入缓存。

```cpp
// MMKV.cpp
// 改进后的读取数据函数，加入缓存机制
size_t MMKV::readData(const std::string& key, void* value, size_t size) const {
    pthread_mutex_lock(&m_mutex);
    try {
        if (getFromCache(key, value, size)) {
            return size;
        }

        char* dataPtr = static_cast<char*>(m_ptr) + m_header->dataOffset;
        while (dataPtr < static_cast<char*>(m_ptr) + m_header->usedSize) {
            uint32_t keyLength;
            memcpy(&keyLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            std::string currentKey(dataPtr, keyLength);
            dataPtr += keyLength;
            uint32_t valueLength;
            memcpy(&valueLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            if (currentKey == key) {
                if (size >= valueLength) {
                    memcpy(value, dataPtr, valueLength);
                    // 将读取的数据存入缓存
                    putIntoCache(key, value, valueLength);
                    pthread_mutex_unlock(&m_mutex);
                    return valueLength;
                }
            }
            dataPtr += valueLength;
        }
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
    return 0;
}
```

通过缓存机制，MMKV能够有效减少对内存映射文件的访问次数，特别是对于频繁访问的数据，显著提升了读写性能。

### 七、数据版本管理与兼容性优化

#### 7.1 数据版本号机制

MMKV通过在 `MMKVHeader` 中记录版本号来管理数据的版本。当MMKV的存储格式发生变化时，通过版本号来判断是否需要进行数据迁移或兼容处理。

```cpp
// MMKVHeader.cpp
// MMKV 数据存储的头部信息结构体
struct MMKVHeader {
    // 魔数，用于标识文件是 MMKV 文件
    uint32_t magic;
    // MMKV 的版本号
    uint32_t version;
    // 数据存储区域的偏移量
    uint32_t dataOffset;
    // 已使用的内存大小
    uint32_t usedSize;
};

// MMKV.cpp
// MMKV 初始化时检查版本号
MMKV::MMKV(const std::string& path, size_t size) {
    //...（文件打开、内存映射等操作）

    // 初始化头部信息
    m_header = static_cast<MMKVHeader*>(m_ptr);
    if (m_header->magic != MMKV_MAGIC) {
        // 魔数不匹配，说明文件不是 MMKV 文件，重新初始化头部信息
        m_header->magic = MMKV_MAGIC;
        m_header->version = MMKV_VERSION;
        m_header->dataOffset = sizeof(MMKVHeader);
        m_header->usedSize = 0;
    } else if (m_header->version < MMKV_VERSION) {
        // 版本号低于当前版本，进行数据迁移
        performDataMigration();
    }
    m_usedSize = m_header->usedSize;
    // 初始化互斥锁
    pthread_mutex_init(&m_mutex, nullptr);
}
```

在MMKV初始化时，会检查文件头部的魔数和版本号。如果魔数不匹配，说明该文件不是MMKV文件，会重新初始化头部信息；如果版本号低于当前版本，则调用 `performDataMigration` 函数进行数据迁移，以保证数据的兼容性和正确性。

#### 7.2 数据迁移策略

当检测到数据版本过低时，MMKV需要将旧版本的数据迁移到新版本的存储格式。数据迁移过程需要保证数据的完整性和一致性。

```cpp
// MMKV.cpp
// 数据迁移函数
void MMKV::performDataMigration() {
    std::vector<BatchWriteData> migratedData;
    char* dataPtr = static_cast<char*>(m_ptr) + m_header->dataOffset;
    while (dataPtr < static_cast<char*>(m_ptr) + m_header->usedSize) {
        uint32_t keyLength;
        memcpy(&keyLength, dataPtr, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        std::string currentKey(dataPtr, keyLength);
        dataPtr += keyLength;
        uint32_t valueLength;
        memcpy(&valueLength, dataPtr, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        void* value = dataPtr;
        dataPtr += valueLength;

        // 这里可以根据旧版本和新版本存储格式的差异，对数据进行转换
        // 例如，旧版本可能使用了不同的编码方式，需要进行解码和重新编码
        BatchWriteData newData;
        newData.key = currentKey;
        newData.value = value;
        newData.size = valueLength;
        migratedData.push_back(newData);
    }

    // 清空原有数据
    m_header->usedSize = sizeof(MMKVHeader);
    m_usedSize = m_header->usedSize;
    m_header->dataOffset = sizeof(MMKVHeader);

    // 将迁移后的数据重新写入
    batchWrite(migratedData);
    // 更新版本号
    m_header->version = MMKV_VERSION;
}
```

在 `performDataMigration` 函数中，首先遍历旧版本的数据存储区域，将每个数据项的信息提取出来。然后根据新旧版本存储格式的差异，对数据进行转换（如编码方式的调整等）。接着清空原有数据，将迁移后的数据通过 `batchWrite` 函数重新写入内存映射文件，并更新版本号。这样就完成了数据从旧版本到新版本的迁移，保证了MMKV在存储格式更新时数据的兼容性和可用性。

### 八、与Android系统交互的优化

#### 8.1 内存映射与Android系统内存管理协同

Android系统有着自己的内存管理机制，MMKV在使用内存映射文件时，需要与Android系统的内存管理进行良好的协同。在Android中，当系统内存紧张时，会触发内存回收机制。MMKV通过合理设置内存映射的属性，使得在内存回收时能够尽量减少对自身数据的影响。

```cpp
// MMKV.cpp
// 进行内存映射
m_ptr = mmap(nullptr, m_size, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
if (m_ptr == MAP_FAILED) {
    close(m_fd);
    // 内存映射失败，抛出异常
    throw std::runtime_error("mmap failed");
}
```

在进行 `mmap` 时，使用 `MAP_SHARED` 标志，这使得内存映射区域是共享的，并且对映射区域的修改会反映回文件。这样当Android系统进行内存回收时，MMKV映射的内存区域如果被回收，其数据的修改已经同步到文件中，下次访问时可以从文件中重新映射恢复数据，保证了数据的持久性和一致性。

同时，MMKV会根据Android系统的内存状态，合理调整自身的内存使用。例如，当检测到系统内存紧张时，MMKV可以适当缩小缓存大小，释放一部分内存空间，以配合系统的内存管理。

```cpp
// MMKV.cpp
// 检查系统内存状态并调整缓存大小的函数
void MMKV::adjustCacheAccordingToSystemMemory() {
    // 获取系统可用内存（这里只是示例，实际获取方式可能更复杂）
    long availableMemory = getAvailableSystemMemory();
    if (availableMemory < THRESHOLD_LOW_MEMORY) {
        // 系统内存紧张，缩小缓存大小
        while (m_cache.size() > CACHE_SIZE_LIMIT_LOW_MEMORY) {
            auto oldest = std::min_element(m_cache.begin(), m_cache.end(),
                                            {
                                               return a.lastAccessTime < b.lastAccessTime;
                                           });
            m_cache.erase(oldest);
        }
    }
}

// 调用示例，在合适的时机调用，如在数据读写操作前后检查
size_t MMKV::readData(const std::string& key, void* value, size_t size) const {
    pthread_mutex_lock(&m_mutex);
    try {
        adjustCacheAccordingToSystemMemory();
        if (getFromCache(key, value, size)) {
            return size;
        }
        //...（后续读取操作）
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
    return 0;
}
```

上述代码中， `adjustCacheAccordingToSystemMemory` 函数通过获取系统可用内存（实际实现中需要使用Android系统提供的相关API准确获取），当系统内存低于一定阈值时，缩小缓存大小，释放内存。在 `readData` 等数据读写函数中，会在操作前后调用该函数，以保证MMKV的内存使用符合系统的整体内存管理策略。

#### 8.2 Android 生命周期与MMKV内存管理

MMKV需要感知Android应用的生命周期，在应用不同的生命周期阶段进行相应的内存管理操作，以优化性能和资源占用。

```java
// MMKVHelper.java (假设的Java辅助类，用于与MMKV的C++部分交互)
import android.app.Application;
import android.content.ComponentCallbacks2;
import android.content.res.Configuration;

public class MMKVHelper {
    private static MMKVHelper instance;
    private MMKV mmkv;

    private MMKVHelper(Application application) {
        // 初始化MMKV，传入文件路径等参数
        mmkv = MMKV.defaultMMKV();

        application.registerComponentCallbacks(new ComponentCallbacks2() {
            @Override
            public void onTrimMemory(int level) {
                // 根据系统的内存清理级别，进行相应的MMKV内存管理操作
                if (level >= ComponentCallbacks2.TRIM_MEMORY_MODERATE) {
                    // 内存压力较大，释放部分可释放的资源，如清理缓存
                    mmkv.clearCache();
                }
            }

            @Override
```

### 九、内存管理的错误处理与稳定性优化

#### 9.1 系统调用错误处理

在MMKV的内存管理过程中，涉及大量的系统调用，如 `open` 、 `mmap` 、 `ftruncate` 等。这些系统调用可能会因为各种原因（如文件权限不足、磁盘空间满等）而失败，MMKV通过完善的错误处理机制来保证程序的稳定性。

```cpp
// MMKV.cpp
// MMKV 初始化时的文件打开操作及错误处理
MMKV::MMKV(const std::string &path, size_t size) {
    // 打开文件，如果文件不存在则创建
    m_fd = open(path.c_str(), O_RDWR | O_CREAT, 0666);
    if (m_fd < 0) {
        // 使用 errno 获取具体的错误码，并转换为字符串描述
        char errorBuf[128];
        const char* errorMsg = strerror_r(errno, errorBuf, sizeof(errorBuf));
        // 构造包含详细错误信息的异常信息
        std::string errInfo = "open file failed: ";
        errInfo += errorMsg;
        // 抛出带有详细信息的运行时异常
        throw std::runtime_error(errInfo);
    }

    // 获取文件大小
    struct stat st;
    if (fstat(m_fd, &st) == 0 && st.st_size > 0) {
        m_size = st.st_size;
    } else {
        // 文件大小获取失败，同样进行详细的错误处理
        char errorBuf[128];
        const char* errorMsg = strerror_r(errno, errorBuf, sizeof(errorBuf));
        std::string errInfo = "fstat file failed: ";
        errInfo += errorMsg;
        throw std::runtime_error(errInfo);

        m_size = size;
        // 调整文件大小
        if (ftruncate(m_fd, m_size) < 0) {
            char errorBuf[128];
            const char* errorMsg = strerror_r(errno, errorBuf, sizeof(errorBuf));
            std::string errInfo = "ftruncate file failed: ";
            errInfo += errorMsg;
            close(m_fd);
            throw std::runtime_error(errInfo);
        }
    }

    // 进行内存映射
    m_ptr = mmap(nullptr, m_size, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
    if (m_ptr == MAP_FAILED) {
        char errorBuf[128];
        const char* errorMsg = strerror_r(errno, errorBuf, sizeof(errorBuf));
        std::string errInfo = "mmap failed: ";
        errInfo += errorMsg;
        close(m_fd);
        throw std::runtime_error(errInfo);
    }

    // 后续初始化操作...
}
```

在上述代码中，每次系统调用失败后，都会通过 `strerror_r` 函数获取具体的错误码描述，并将其融入到异常信息中再抛出。这样开发者在捕获异常时，能够清晰地了解到是哪个系统调用出现问题以及具体的错误原因，方便进行问题排查和修复，增强了MMKV在面对系统调用错误时的稳定性。

#### 9.2 数据一致性保障

在多线程环境下或者数据读写过程中，可能会出现数据不一致的情况。MMKV通过多种方式来保障数据的一致性。除了前面提到的互斥锁机制，在数据写入和读取过程中，还会进行数据校验。

```cpp
// MMKV.cpp
// 写入数据时添加简单的数据校验（示例：校验数据长度是否合理）
void MMKV::writeData(const void *value, size_t size, const std::string &key) {
    pthread_mutex_lock(&m_mutex);
    try {
        // 简单校验数据长度，避免写入非法数据
        if (size > MAX_ALLOWED_DATA_SIZE) {
            throw std::runtime_error("Data size exceeds the allowed limit");
        }

        ensureSize(size + key.size() + sizeof(uint32_t) * 2);

        char *dataPtr = static_cast<char *>(m_ptr) + m_header->dataOffset;
        // 写入 key 的长度
        uint32_t keyLength = static_cast<uint32_t>(key.size());
        memcpy(dataPtr, &keyLength, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        // 写入 key
        memcpy(dataPtr, key.c_str(), keyLength);
        dataPtr += keyLength;
        // 写入数据的长度
        uint32_t valueLength = static_cast<uint32_t>(size);
        memcpy(dataPtr, &valueLength, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        // 写入数据
        memcpy(dataPtr, value, size);

        m_header->usedSize += size + key.size() + sizeof(uint32_t) * 2;
        m_usedSize = m_header->usedSize;
        m_header->dataOffset += size + key.size() + sizeof(uint32_t) * 2;
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
}

// 读取数据时进行数据完整性校验（示例：校验数据长度是否与记录一致）
size_t MMKV::readData(const std::string &key, void *value, size_t size) const {
    pthread_mutex_lock(&m_mutex);
    try {
        char *dataPtr = static_cast<char *>(m_ptr) + m_header->dataOffset;
        while (dataPtr < static_cast<char *>(m_ptr) + m_header->usedSize) {
            uint32_t keyLength;
            memcpy(&keyLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            std::string currentKey(dataPtr, keyLength);
            dataPtr += keyLength;
            uint32_t valueLength;
            memcpy(&valueLength, dataPtr, sizeof(uint32_t));
            dataPtr += sizeof(uint32_t);
            if (currentKey == key) {
                // 校验读取到的数据长度是否合理
                if (valueLength > size) {
                    throw std::runtime_error("Insufficient buffer size for data");
                }
                memcpy(value, dataPtr, valueLength);
                pthread_mutex_unlock(&m_mutex);
                return valueLength;
            }
            dataPtr += valueLength;
        }
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
    return 0;
}
```

在 `writeData` 函数中，对要写入的数据大小进行了限制校验，防止写入过大的非法数据；在 `readData` 函数中，会校验读取到的数据长度是否与记录一致，以及是否超过了提供的缓冲区大小。通过这些校验措施，在一定程度上保障了数据的一致性，避免因数据错误导致程序出现异常或数据混乱。

### 十、内存管理的性能监控与调优

#### 10.1 性能指标采集

MMKV为了更好地了解自身内存管理的性能状况，会采集一些关键的性能指标。这些指标包括内存分配次数、内存释放次数、数据读写耗时等。

```cpp
// MMKV.cpp
// MMKV 类中新增性能指标相关成员变量
size_t m_memoryAllocationCount; // 内存分配次数
size_t m_memoryDeallocationCount; // 内存释放次数
std::chrono::nanoseconds m_totalReadTime; // 总读取时间
std::chrono::nanoseconds m_totalWriteTime; // 总写入时间

// 内存分配操作时更新指标（示例：在 ensureSize 中涉及内存扩展可视为一种内存分配）
void MMKV::ensureSize(size_t needSize) {
    if (m_usedSize + needSize <= m_size) {
        return;
    }

    // 计算新的文件大小，每次扩展为原来的 2 倍
    size_t newSize = std::max(m_size * 2, m_usedSize + needSize);
    // 调整文件大小
    if (ftruncate(m_fd, newSize) < 0) {
        throw std::runtime_error("ftruncate file failed");
    }

    // 重新映射内存，将新扩展的内存区域映射进来
    auto start = std::chrono::high_resolution_clock::now();
    void *newPtr = mremap(m_ptr, m_size, newSize, MREMAP_MAYMOVE);
    auto end = std::chrono::high_resolution_clock::now();
    if (newPtr == MAP_FAILED) {
        throw std::runtime_error("mremap failed");
    }

    m_memoryAllocationCount++;
    m_totalReadTime += std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);

    m_ptr = newPtr;
    m_size = newSize;
    m_header = static_cast<MMKVHeader *>(m_ptr);
}

// 数据写入操作时更新指标
void MMKV::writeData(const void *value, size_t size, const std::string &key) {
    pthread_mutex_lock(&m_mutex);
    try {
        auto start = std::chrono::high_resolution_clock::now();

        ensureSize(size + key.size() + sizeof(uint32_t) * 2);

        char *dataPtr = static_cast<char *>(m_ptr) + m_header->dataOffset;
        // 写入 key 的长度
        uint32_t keyLength = static_cast<uint32_t>(key.size());
        memcpy(dataPtr, &keyLength, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        // 写入 key
        memcpy(dataPtr, key.c_str(), keyLength);
        dataPtr += keyLength;
        // 写入数据的长度
        uint32_t valueLength = static_cast<uint32_t>(size);
        memcpy(dataPtr, &valueLength, sizeof(uint32_t));
        dataPtr += sizeof(uint32_t);
        // 写入数据
        memcpy(dataPtr, value, size);

        m_header->usedSize += size + key.size() + sizeof(uint32_t) * 2;
        m_usedSize = m_header->usedSize;
        m_header->dataOffset += size + key.size() + sizeof(uint32_t) * 2;

        auto end = std::chrono::high_resolution_clock::now();
        m_totalWriteTime += std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);
    } catch (...) {
        pthread_mutex_unlock(&m_mutex);
        // 发生异常，重新抛出
        throw;
    }
    pthread_mutex_unlock(&m_mutex);
}
```

上述代码在内存分配（如 `ensureSize` 函数中的内存扩展操作）和数据写入操作时，记录了操作次数和耗时。通过采集这些性能指标，能够直观地了解MMKV内存管理在不同操作下的性能表现，为后续的性能分析和优化提供数据支持。

#### 10.2 性能调优策略

基于采集到的性能指标，MMKV可以制定相应的性能调优策略。例如，如果发现内存分配次数过多，可能是内存分配粒度不合理，可以尝试调整内存扩展的策略；如果数据读写耗时较长，可以进一步优化缓存机制或数据编码方式。

```cpp
// MMKV.cpp
// 根据性能指标调整内存扩展策略的示例函数
void MMKV::optimizeMemoryAllocation() {
    // 如果内存分配次数在一定时间内过高
    if (m_memoryAllocationCount > ALLOCATION_THRESHOLD) {
        // 调整内存扩展倍数，例如从原来的 2 倍调整为 1.5 倍
        MEMORY_EXTENSION_FACTOR = 1.5;
    }
}

// 调用示例，在合适的时机调用，如定期检查或在性能明显下降时
void MMKV::periodicPerformanceCheck() {
    optimizeMemoryAllocation();
    // 后续还可以添加对读写耗时等指标的分析和优化操作
}
```

在 `optimizeMemoryAllocation` 函数中，通过判断内存分配次数是否超过阈值，来决定是否调整内存扩展倍数。在 `periodicPerformanceCheck` 函数中，可以定期调用性能优化相关函数，根据性能指标对MMKV的内存管理策略进行动态调整，以达到最佳的性能表现。

