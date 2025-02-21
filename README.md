# Лабораторная работа 2
```cpp
// app.cpp
#include <iostream>
#include <cstdlib>
#include <ctime>
#include <chrono>
#include "cache.h"
#include <iomanip>


string getCurrentTime() {
    auto now = chrono::system_clock::now();
    auto in_time_t = chrono::system_clock::to_time_t(now);
    tm tm;
    localtime_s(&tm, &in_time_t);
    stringstream ss;
    ss << put_time(&tm, "%Y-%m-%d %X");
    return ss.str();
}

void setLogging(bool enable) {
    g_loggingEnabledApp = enable;
}

const size_t FILE_SIZE = 8 * 1024 * 100;
const size_t SECTOR_SIZE = 4096;
const size_t ELEMENT_SIZE = sizeof(int);
const size_t NUM_ELEMENTS = FILE_SIZE / ELEMENT_SIZE;

bool get_element(int fd, size_t index, int &value, void *alignedBuffer);

bool set_element(int fd, size_t index, int value, void *alignedBuffer);

bool swap_elements(int fd, size_t index1, size_t index2, void *alignedBuffer);

bool create_file(const char *path) {
    
    int fd = lab2_open(path);
    if (fd == -1) {
        return false;
    }

    srand(static_cast<unsigned>(time(nullptr)));

    for (size_t i = 0; i < NUM_ELEMENTS; ++i) {
        int value = rand();
        if (lab2_write(fd, &value, ELEMENT_SIZE, i*sizeof(value)) != static_cast<ssize_t>(ELEMENT_SIZE)) {
            lab2_close(fd);
            return false;
        }
    }

    if (lab2_fsync(fd) != 0) {
        lab2_close(fd);
        return false;
    }
    if (lab2_close(fd) != 0) {
        return false;
    }
    return true;
}

bool swap_elements(int fd, size_t index1, size_t index2, void *alignedBuffer) {
    if (index1 == index2) {
        return true;
    }
    int val1, val2;

    if (!get_element(fd, index1, val1, alignedBuffer)) {
        return false;
    }

    if (!get_element(fd, index2, val2, alignedBuffer)) {
        return false;
    }

    if (val1 == val2) {
        return true;
    }

    if (!set_element(fd, index2, val1, alignedBuffer)) {
        return false;
    }

    if (!set_element(fd, index1, val2, alignedBuffer)) {
        return false;
    }

    return true;
}

bool get_element(int fd, size_t index, int &value, void *alignedBuffer) {
    size_t offset = (index * ELEMENT_SIZE) / SECTOR_SIZE * SECTOR_SIZE;
    size_t alignedSize = SECTOR_SIZE;

    if (lab2_lseek(fd, offset, SEEK_SET) == -1) {
        return false;
    }
    if (lab2_read(fd, alignedBuffer, alignedSize) != static_cast<ssize_t>(alignedSize)) {
        return false;
    }
    size_t offsetInBuffer = (index * ELEMENT_SIZE) % SECTOR_SIZE;
    value = *(int *)((char *)alignedBuffer + offsetInBuffer);
    return true;
}

bool set_element(int fd, size_t index, int value, void *alignedBuffer) {
    size_t offset = (index * ELEMENT_SIZE) / SECTOR_SIZE * SECTOR_SIZE;
    size_t alignedSize = SECTOR_SIZE;

    if (lab2_lseek(fd, offset, SEEK_SET) == -1) {
        return false;
    }
    if (lab2_read(fd, alignedBuffer, alignedSize) != static_cast<ssize_t>(alignedSize)) {
        return false;
    }

    size_t offsetInBuffer = (index * ELEMENT_SIZE) % SECTOR_SIZE;
    *(int *)((char *)alignedBuffer + offsetInBuffer) = value;

    if (lab2_lseek(fd, offset, SEEK_SET) == -1) {
        return false;
    }
    if (lab2_write(fd, alignedBuffer, alignedSize, offset) != static_cast<ssize_t>(alignedSize)) {
        return false;
    }

    return true;
}

bool print_first_n(int fd, int n, void *alignedBuffer) {
    cout << "[print_first_n] Printing first " << n << " elements:" << endl;
    for (int i = 0; i < n; ++i) {
        int value;
        if (!get_element(fd, i, value, alignedBuffer)) {
            return false;
        }
        cout << value << " ";
    }
    cout << endl;
    return true;
}

bool verify_sorted(int fd, void *alignedBuffer) {
    int previous = 0;
    for (size_t i = 0; i < NUM_ELEMENTS; ++i) {
        int current;
        if (!get_element(fd, i, current, alignedBuffer)) {
            return false;
        }
        if (i > 0 && current < previous) {
            return false;
        }
        previous = current;
    }
    return true;
}

int partition(int fd, int low, int high, void *alignedBuffer) {
    int pivot;
    if (!get_element(fd, high, pivot, alignedBuffer)) {
        return -1;
    }

    int i = low - 1;

    for (int j = low; j <= high - 1; j++) {
        int current;
        if (!get_element(fd, j, current, alignedBuffer)) {
            return -1;
        }
        if (current < pivot) {
            ++i;
            if (i != j) {
                if (!swap_elements(fd, i, j, alignedBuffer)) {
                    return -1;
                }
            }
        }
    }

    if (!swap_elements(fd, i + 1, high, alignedBuffer)) {
        return -1;
    }

    return i + 1;
}

bool quicksort(int fd, int low, int high, void *alignedBuffer) {
    if (low < high) {
        int pi = partition(fd, low, high, alignedBuffer);
        if (pi == -1) {
            return false;
        }

        if (!quicksort(fd, low, pi - 1, alignedBuffer)) {
            return false;
        }
        if (!quicksort(fd, pi + 1, high, alignedBuffer)) {
            return false;
        }
    }
    return true;
}

int main() {
    const char *file_path = "data.bin";
    cout << "[main] " << getCurrentTime() << "Starting program. File path: " << file_path << endl;

    if (!create_file(file_path)) {
        return EXIT_FAILURE;
    }

    int fd = lab2_open(file_path);
    if (fd == -1) {
        return EXIT_FAILURE;
    }

    void *alignedBuffer = _aligned_malloc(SECTOR_SIZE, SECTOR_SIZE);
    if (!alignedBuffer) {
        return EXIT_FAILURE;
    }

    print_first_n(fd, 10, alignedBuffer)

    if (!quicksort(fd, 0, static_cast<int>(NUM_ELEMENTS - 1), alignedBuffer)) {
        _aligned_free(alignedBuffer);
        lab2_close(fd)
        return EXIT_FAILURE;
    }
    cout << "[main] " << getCurrentTime() << "sort completed: " << endl;

    print_first_n(fd, 10, alignedBuffer)

    if (verify_sorted(fd, alignedBuffer)) {
        cout << "[main] File has been sorted correctly." << endl;
    }

    if (lab2_fsync(fd) != 0) {
        _aligned_free(alignedBuffer);
        lab2_close(fd)
        return EXIT_FAILURE;
    }
    if (lab2_close(fd) != 0) {
        _aligned_free(alignedBuffer);
        return EXIT_FAILURE;
    }
    _aligned_free(alignedBuffer);
    cout << "[main] " << getCurrentTime() << "all done " << endl;

    return EXIT_SUCCESS;
}
```

```cpp
#include <windows.h>
#include <unordered_map>
#include <list>
#include <iostream>
#include <iomanip>
#include <ctime>
#include <locale>
#include <codecvt>
#include <sstream>
#include "cache.h"
#include <bits/algorithmfwd.h>
#include <cstdint>

std::string getCurrentTimeCache() {
    auto now = std::time(nullptr);
    std::tm tm;
    localtime_s(&tm, &now);
    std::stringstream ss;
    ss << std::put_time(&tm, "%Y-%m-%d %H:%M:%S");
    return ss.str();
}

const size_t PAGE_SIZE = 4096;
const size_t NUM_OF_BLOCKS = 100;

class BlockCache {
public:
    static BlockCache &getInstance(size_t cacheSize = NUM_OF_BLOCKS) {
        static BlockCache instance(cacheSize);
        return instance;
    }

    bool read(int fd, void *buf, size_t count, size_t offset);
    bool write(int fd, const void *buf, size_t count, size_t offset);
    void sync(int fd);
    void close(int fd);

private:
    BlockCache(size_t cacheSize) : cacheSize(cacheSize), alignedBuffer(_aligned_malloc(PAGE_SIZE, PAGE_SIZE)) {
        if (!alignedBuffer) {
            throw std::bad_alloc();
        }
    }

    ~BlockCache() {
        if (alignedBuffer) {
            _aligned_free(alignedBuffer);
        }
    }

    struct CacheBlock {
        void *data;
        int fd;
        size_t offset;
        bool dirty;
        size_t frequency;

        CacheBlock() : data(_aligned_malloc(PAGE_SIZE, PAGE_SIZE)), fd(-1), offset(0), dirty(false), frequency(0) {}
    };

    size_t cacheSize;
    std::list<CacheBlock> cache;
    std::unordered_map<int, std::unordered_map<size_t, std::list<CacheBlock>::iterator>> cacheMap;
    void *alignedBuffer;

    void evict();
    void writeBlockToDisk(const CacheBlock &block);
};

bool BlockCache::read(int fd, void *buf, size_t count, size_t offset) {
    size_t alignedOffset = (offset / PAGE_SIZE) * PAGE_SIZE;
    auto &fdMap = cacheMap[fd];
    auto offsetIt = fdMap.find(alignedOffset);
    if (offsetIt != fdMap.end()) {
        memcpy(buf, static_cast<char*>(offsetIt->second->data) + (offset - alignedOffset), count);
        offsetIt->second->frequency++;
        return true;
    }

    HANDLE handle = reinterpret_cast<HANDLE>(fd);
    LARGE_INTEGER li;
    li.QuadPart = alignedOffset;
    SetFilePointerEx(handle, li, NULL, FILE_BEGIN);

    DWORD bytesRead = 0;
    if (!ReadFile(handle, alignedBuffer, PAGE_SIZE, &bytesRead, NULL)) {
        return false;
    }

    memcpy(buf, static_cast<char*>(alignedBuffer) + (offset - alignedOffset), count);

    if (cache.size() >= cacheSize) evict();
    CacheBlock newBlock;
    memcpy(newBlock.data, alignedBuffer, PAGE_SIZE);
    newBlock.fd = fd;
    newBlock.offset = alignedOffset;
    newBlock.dirty = false;
    newBlock.frequency = 1;
    cache.push_front(newBlock);
    cacheMap[fd][alignedOffset] = cache.begin();

    return true;
}

bool BlockCache::write(int fd, const void *buf, size_t count, size_t offset) {
    size_t alignedOffset = (offset / PAGE_SIZE) * PAGE_SIZE;
    auto &fdMap = cacheMap[fd];
    auto offsetIt = fdMap.find(alignedOffset);
    if (offsetIt != fdMap.end()) {
        memcpy(static_cast<char*>(offsetIt->second->data) + (offset - alignedOffset), buf, count);
        offsetIt->second->dirty = true;
        offsetIt->second->frequency++;
        return true;
    }

    HANDLE handle = reinterpret_cast<HANDLE>(fd);
    LARGE_INTEGER li;
    li.QuadPart = alignedOffset;
    SetFilePointerEx(handle, li, NULL, FILE_BEGIN);
    DWORD bytesRead = 0;
    if (!ReadFile(handle, alignedBuffer, PAGE_SIZE, &bytesRead, NULL)) {
        return false;
    }

    memcpy(static_cast<char*>(alignedBuffer) + (offset - alignedOffset), buf, count);

    if (cache.size() >= cacheSize) evict();
    CacheBlock newBlock;
    memcpy(newBlock.data, alignedBuffer, PAGE_SIZE);
    newBlock.fd = fd;
    newBlock.offset = alignedOffset;
    newBlock.dirty = true;
    newBlock.frequency = 1;
    cache.push_front(newBlock);
    cacheMap[fd][alignedOffset] = cache.begin();

    return true;
}

void BlockCache::sync(int fd) {
    auto fdIt = cacheMap.find(fd);
    if (fdIt != cacheMap.end()) {
        for (auto &offsetIt : fdIt->second) {
            if (offsetIt.second->dirty) {
                writeBlockToDisk(*offsetIt.second);
                offsetIt.second->dirty = false;
            }
            _aligned_free(offsetIt.second->data);
            cache.erase(offsetIt.second);
        }
        cacheMap.erase(fdIt);
    }
}

void BlockCache::close(int fd) {
    sync(fd);
}

void BlockCache::evict()
{
    if (!cache.empty()) {
        auto it = cache.begin();
        if (it->dirty) {
            writeBlockToDisk(*it);
        }
        cacheMap[it->fd].erase(it->offset);
        _aligned_free(it->data);
        cache.erase(it);
    }
}

void BlockCache::writeBlockToDisk(const CacheBlock &block) {
    HANDLE handle = reinterpret_cast<HANDLE>(block.fd);

    LARGE_INTEGER li;
    li.QuadPart = block.offset;
    SetFilePointerEx(handle, li, NULL, FILE_BEGIN);

    DWORD bytesWritten = 0;
    WriteFile(handle, block.data, PAGE_SIZE, &bytesWritten, NULL);
}

int lab2_open(const char *path) {
    HANDLE hFile = CreateFileA(
            path,
            GENERIC_READ | GENERIC_WRITE,
            0,
            NULL,
            OPEN_ALWAYS,
            FILE_FLAG_NO_BUFFERING | FILE_FLAG_WRITE_THROUGH,
            NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        return -1;
    }
    return reinterpret_cast<intptr_t>(hFile);
}

ssize_t lab2_read(int fd, void *buf, size_t count) {
    HANDLE handle = reinterpret_cast<HANDLE>(fd);

    LARGE_INTEGER currentPos;
    if (!SetFilePointerEx(handle, {0}, &currentPos, FILE_CURRENT)) {
        return -1;
    }

    size_t offset = static_cast<size_t>(currentPos.QuadPart);

    if (BlockCache::getInstance().read(fd, buf, count, offset)) {
        LARGE_INTEGER li;
        li.QuadPart = offset + count;
        if (!SetFilePointerEx(handle, li, NULL, FILE_BEGIN)) {
            return -1;
        }

        return static_cast<ssize_t>(count);
    }

    return -1;
}

ssize_t lab2_write(int fd, const void *buf, size_t count, size_t offset) {
    HANDLE handle = reinterpret_cast<HANDLE>(fd);

    if (BlockCache::getInstance().write(fd, buf, count, offset)) {
        return static_cast<ssize_t>(count);
    }

    return -1;
}

int lab2_close(int fd) {
    HANDLE handle = reinterpret_cast<HANDLE>(fd);
    if (handle != INVALID_HANDLE_VALUE) {
        BlockCache::getInstance().close(fd);
        if (CloseHandle(handle)) {
            return 0;
        } else {
            return -1;
        }
    }
    return -1;
}

int64_t lab2_lseek(int fd, int64_t offset, int whence) {
    HANDLE handle = reinterpret_cast<HANDLE>(fd);
    LARGE_INTEGER li;
    li.QuadPart = offset;
    if (!SetFilePointerEx(handle, li, NULL, whence)) {
        return -1;
    }

    LARGE_INTEGER newPos;
    if (!SetFilePointerEx(handle, {0}, &newPos, FILE_CURRENT)) {
        return -1;
    }

    return static_cast<int64_t>(newPos.QuadPart);
}

int lab2_fsync(int fd) {
    HANDLE handle = reinterpret_cast<HANDLE>(fd);
    BlockCache::getInstance().sync(fd);
    if (FlushFileBuffers(handle)) {
        return 0;
    } else {
        return -1;
    }
}
```
<!--![With Cache](https://github.com/user-attachments/assets/813aeba8-4236-433d-b900-5a02395c5bfb)


![Without Cache](https://github.com/user-attachments/assets/f725ab60-91cb-40ab-8a77-dc909509310a)-->

![image](https://github.com/user-attachments/assets/8071abdc-b228-4a00-869a-57e698289b23)
![image](https://github.com/user-attachments/assets/a75bb48c-30f2-4dc8-9dc3-5bff950fe2f8)


![image](https://github.com/user-attachments/assets/e8ec11e2-ccd9-45bb-9b76-0965797fd69b)
![image](https://github.com/user-attachments/assets/b4aa8985-723d-44b2-9523-c8bd90bd121d)




Не трудно заметить, что кеш сильно ускоряет рабоету программы, даже со столь неэффективным алгоритмом замены блоков как MRU.
Это происходит по тем причинам, что:  
    - Меньше операций с диском: без кеша каждый вызов get_element() требует похода на диск  
    - Сохранение часто используемых данных  
    - Если кеш записывает данные, он не делает это сразу, вместо этого изменения сохраняются в кеше, а затем записываются одним большим блоком
