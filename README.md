# Лабораторная работа 2
```cpp
// app.cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
#include <cstdlib>
#include <ctime>
#include <chrono>
#include "cache.h"
#include <iomanip>

using namespace std;

const size_t FILE_SIZE = 8 * 1024 * 100;
const size_t SECTOR_SIZE = 4096;
const size_t ELEMENT_SIZE = sizeof(int);
const size_t NUM_ELEMENTS = FILE_SIZE / ELEMENT_SIZE;

string getCurrentTime() {
    auto now = chrono::system_clock::now();
    auto in_time_t = chrono::system_clock::to_time_t(now);
    tm tm;
    localtime_s(&tm, &in_time_t);
    stringstream ss;
    ss << put_time(&tm, "%Y-%m-%d %X");
    return ss.str();
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

bool swap_elements(int fd, size_t index1, size_t index2, void *alignedBuffer) {
    if (index1 == index2) {
        return true;
    }

    int val1, val2;
    if (!get_element(fd, index1, val1, alignedBuffer)) return false;
    if (!get_element(fd, index2, val2, alignedBuffer)) return false;

    if (val1 == val2) return true;

    if (!set_element(fd, index2, val1, alignedBuffer)) return false;
    if (!set_element(fd, index1, val2, alignedBuffer)) return false;

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

bool quicksort(int fd, int low, int high, void *alignedBuffer) {
    if (low >= high) return true;

    int pivot;
    if (!get_element(fd, high, pivot, alignedBuffer)) return false;

    int i = low - 1;
    for (int j = low; j <= high - 1; j++) {
        int current;
        if (!get_element(fd, j, current, alignedBuffer)) return false;
        if (current < pivot) {
            ++i;
            if (!swap_elements(fd, i, j, alignedBuffer)) return false;
        }
    }

    if (!swap_elements(fd, i + 1, high, alignedBuffer)) return false;

    if (!quicksort(fd, low, i, alignedBuffer)) return false;
    if (!quicksort(fd, i + 2, high, alignedBuffer)) return false;

    return true;
}


void sort_part(int fd, int start, int end, void *alignedBuffer) {
    quicksort(fd, start, end, alignedBuffer);
}

bool merge_sorted_parts(int fd, int num_threads, int part_size, void *alignedBuffer) {
    cout << "[merge] Объединение отсортированных частей..." << endl;
    
    vector<int> merged(NUM_ELEMENTS);
    int idx = 0;

    for (int i = 0; i < num_threads; i++) {
        int start = i * part_size;
        int end = min(start + part_size, (int)NUM_ELEMENTS);

        for (int j = start; j < end; j++) {
            int value;
            if (!get_element(fd, j, value, alignedBuffer)) return false;
            merged[idx++] = value;
        }
    }

    for (size_t i = 0; i < NUM_ELEMENTS; i++) {
        if (!set_element(fd, i, merged[i], alignedBuffer)) return false;
    }

    return true;
}

int main(int argc, char* argv[]) {
    if (argc != 3) {
        cerr << "Usage: " << argv[0] << " <input_file> <num_threads>" << endl;
        return 1;
    }
    const char *file_path = argv[1];
    cout << "[main] " << getCurrentTime() << " Начало программы. Файл: " << file_path << endl;

    int fd = lab2_open(file_path);
    if (fd == -1) {
        cerr << "[ERROR] Не удалось открыть файл!" << endl;
        return EXIT_FAILURE;
    }

    void *alignedBuffer = _aligned_malloc(SECTOR_SIZE, SECTOR_SIZE);
    if (!alignedBuffer) {
        cerr << "[ERROR] Ошибка выделения памяти!" << endl;
        return EXIT_FAILURE;
    }

    size_t num_threads = std::stoul(argv[2]);
    int part_size = (NUM_ELEMENTS + num_threads - 1) / num_threads;

    vector<thread> threads;

    for (int i = 0; i < num_threads; i++) {
        int start = i * part_size;
        int end = min(start + part_size, (int)NUM_ELEMENTS);
        
        if (start < NUM_ELEMENTS && end > start) {  // Проверка на пустые диапазоны
            threads.emplace_back(sort_part, fd, start, end - 1, alignedBuffer);
        }
    }

    cout << "[main] Ожидание завершения потоков..." << endl;
    for (auto &thread : threads) {
        if (thread.joinable()) {
            thread.join();
        }
    }
    cout << "[main] Все потоки завершены." << endl;

    if (!merge_sorted_parts(fd, num_threads, part_size, alignedBuffer)) {
        _aligned_free(alignedBuffer);
        lab2_close(fd);
        return EXIT_FAILURE;
    }

    cout << "[main] " << getCurrentTime() << " Сортировка завершена." << endl;

    if (verify_sorted(fd, alignedBuffer)) {
        cout << "[main] Файл отсортирован корректно." << endl;
    }

    if (lab2_fsync(fd) != 0) {
        _aligned_free(alignedBuffer);
        lab2_close(fd);
        return EXIT_FAILURE;
    }
    if (lab2_close(fd) != 0) {
        _aligned_free(alignedBuffer);
        return EXIT_FAILURE;
    }

    _aligned_free(alignedBuffer);
    cout << "[main] " << getCurrentTime() << " Завершение программы." << endl;

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
#include <mutex>
#include "cache.h"

const size_t PAGE_SIZE = 4096;
const size_t NUM_OF_BLOCKS = 100;
std::mutex file_mutex;

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
    std::mutex cache_mutex;

    void evict();
    void writeBlockToDisk(const CacheBlock &block);
};

bool BlockCache::read(int fd, void *buf, size_t count, size_t offset) {
    size_t alignedOffset = (offset / PAGE_SIZE) * PAGE_SIZE;

    std::lock_guard<std::mutex> lock(cache_mutex);

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

    std::lock_guard<std::mutex> lock(cache_mutex);

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
    std::lock_guard<std::mutex> lock(cache_mutex);

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

void BlockCache::evict() {
    std::lock_guard<std::mutex> lock(cache_mutex);  
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
    std::lock_guard<std::mutex> lock(file_mutex);
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
    std::lock_guard<std::mutex> lock(file_mutex);

    if (BlockCache::getInstance().read(fd, buf, count, 0)) {
        return static_cast<ssize_t>(count);
    }
    return -1;
}

ssize_t lab2_write(int fd, const void *buf, size_t count, size_t offset) {
    std::lock_guard<std::mutex> lock(file_mutex);

    if (BlockCache::getInstance().write(fd, buf, count, offset)) {
        return static_cast<ssize_t>(count);
    }
    return -1;
}

int lab2_fsync(int fd) {
    std::lock_guard<std::mutex> lock(file_mutex);
    BlockCache::getInstance().sync(fd);
    return 0;
}

int lab2_close(int fd) {
    std::lock_guard<std::mutex> lock(file_mutex);
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
    std::lock_guard<std::mutex> lock(file_mutex);
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
```
<!--![With Cache](https://github.com/user-attachments/assets/813aeba8-4236-433d-b900-5a02395c5bfb)


![Without Cache](https://github.com/user-attachments/assets/f725ab60-91cb-40ab-8a77-dc909509310a)-->

![image](https://github.com/user-attachments/assets/8071abdc-b228-4a00-869a-57e698289b23)
![image](https://github.com/user-attachments/assets/a75bb48c-30f2-4dc8-9dc3-5bff950fe2f8)


![image](https://github.com/user-attachments/assets/e8ec11e2-ccd9-45bb-9b76-0965797fd69b)
![image](https://github.com/user-attachments/assets/b4aa8985-723d-44b2-9523-c8bd90bd121d)

![image](https://github.com/user-attachments/assets/2b8b7f2d-1f02-42bc-bd06-479764c2a2df)
![image](https://github.com/user-attachments/assets/c9fc5fe6-ad7f-4904-8b22-7ea48950e742)



Не трудно заметить, что кеш сильно ускоряет рабоету программы, даже со столь неэффективным алгоритмом замены блоков как MRU.
Это происходит по тем причинам, что:  
    - Меньше операций с диском: без кеша каждый вызов get_element() требует похода на диск  
    - Сохранение часто используемых данных  
    - Если кеш записывает данные, он не делает это сразу, вместо этого изменения сохраняются в кеше, а затем записываются одним большим блоком
