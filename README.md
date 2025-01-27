# Лабораторная работа 2
```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
#include <unordered_set>
#include <algorithm>
#include <string>
#include <cstring>
#include <windows.h>
#include "lib/block_cache.h"

using namespace std;

mutex mtx;

void process_chunk(const vector<int>& data, size_t start, size_t end, unordered_set<int>& local_unique) {
    for (size_t i = start; i < end; ++i) {
        local_unique.insert(data[i]);
    }
}

vector<int> deduplicate(const vector<int>& data, int num_threads) {
    vector<thread> threads;                   
    vector<unordered_set<int>> thread_results(num_threads); 
    unordered_set<int> global_unique;

    const size_t chunk_size = data.size() / num_threads; 
    size_t remaining = data.size() % num_threads;        
    size_t start = 0;

    for (int i = 0; i < num_threads; ++i) {
        size_t end = start + chunk_size + (i < remaining ? 1 : 0);
        threads.emplace_back(process_chunk, cref(data), start, end, ref(thread_results[i]));
        start = end;
    }

    for (auto& t : threads) {
        t.join();
    }

    for (const auto& local_set : thread_results) {
        lock_guard<mutex> lock(mtx); 
        
        global_unique.insert(local_set.begin(), local_set.end());
    }

    vector<int> result(global_unique.begin(), global_unique.end());
    sort(result.begin(), result.end());

    return result;
}

vector<int> read_numbers(const string& filename) {
    int fd = lab2_open(filename.c_str());
    if (fd < 0) {
        throw runtime_error("Не удалось открыть файл: " + filename);
    }

    vector<int> numbers;
    int num;
    char buffer[sizeof(int)];
    while (lab2_read(fd, buffer, sizeof(buffer)) == sizeof(buffer)) {
        memcpy(&num, buffer, sizeof(int));
        numbers.push_back(num);
    }

    lab2_close(fd);
    return numbers;
}

int main(int argc, char* argv[]) {
    if (argc != 3) {
        cerr << "Использование: " << argv[0] << " <имя_файла> <количество_потоков>\n";
        return 1;
    }

    try {
        string filename = argv[1];
        int num_threads = stoi(argv[2]);

        if (num_threads < 1) {
            throw invalid_argument("Количество потоков должно быть не менее 1");
        }

        auto numbers = read_numbers(filename);
        cout << "Прочитано чисел из файла: " << numbers.size() << "\n";

        auto unique_numbers = deduplicate(numbers, num_threads);

        cout << "Количество уникальных чисел: " << unique_numbers.size() << "\n";
        cout << "Уникальные числа:\n";
        for (int num : unique_numbers) {
            cout << num << " ";
        }
        cout << endl;
    }
    catch (const exception& e) {
        cerr << "Ошибка: " << e.what() << endl;
        return 1;
    }

    return 0;
}
```

```cpp
#include <windows.h>
#include <unordered_map>
#include <list>
#include <vector>
#include <stdexcept>
#include <mutex>
#include <string>
#include <cstdint> 
#include "block_cache.h"

#define BLOCK_SIZE 4096 

using namespace std;

struct CacheBlock {
    vector<char> data; 
    intptr_t offset;   
    bool dirty;        
};

class BlockCache {
private:
    size_t capacity; 
    unordered_map<intptr_t, HANDLE> fileHandles; 
    unordered_map<intptr_t, list<CacheBlock>::iterator> cacheMap;
    list<CacheBlock> cacheList; 
    mutex cacheMutex;           

public:
    explicit BlockCache(size_t cacheCapacity) : capacity(cacheCapacity) {}

    intptr_t lab2_open(const char* path) {
        lock_guard<mutex> lock(cacheMutex);

        HANDLE fileHandle = CreateFileA(path, GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING,
                                        FILE_FLAG_NO_BUFFERING | FILE_FLAG_WRITE_THROUGH, NULL);

        if (fileHandle == INVALID_HANDLE_VALUE) {
            throw runtime_error("Cannot open file.");
        }

        intptr_t fd = reinterpret_cast<intptr_t>(fileHandle);
        fileHandles[fd] = fileHandle;
        return fd;
    }

    int lab2_close(intptr_t fd) {
        lock_guard<mutex> lock(cacheMutex);

        if (fileHandles.find(fd) == fileHandles.end()) {
            return -1; 
        }

        
        lab2_fsync(fd);

        CloseHandle(fileHandles[fd]);
        fileHandles.erase(fd);

        for (auto it = cacheList.begin(); it != cacheList.end();) {
            if (it->offset == fd) {
                it = cacheList.erase(it);
            } else {
                ++it;
            }
        }
        return 0;
    }

    long long lab2_read(intptr_t fd, void* buf, size_t count) {
        lock_guard<mutex> lock(cacheMutex);

        if (fileHandles.find(fd) == fileHandles.end()) {
            return -1; 
        }

        HANDLE fileHandle = fileHandles[fd];
        DWORD bytesRead;
        if (!ReadFile(fileHandle, buf, static_cast<DWORD>(count), &bytesRead, NULL)) {
            return -1; 
        }
        return bytesRead;
    }

    long long slab2_write(intptr_t fd, const void* buf, size_t count) {
        lock_guard<mutex> lock(cacheMutex);

        if (fileHandles.find(fd) == fileHandles.end()) {
            return -1; 
        }

        HANDLE fileHandle = fileHandles[fd];
        DWORD bytesWritten;
        if (!WriteFile(fileHandle, buf, static_cast<DWORD>(count), &bytesWritten, NULL)) {
            return -1; 
        }

        CacheBlock block;
        block.data.assign(static_cast<const char*>(buf), static_cast<const char*>(buf) + count);
        block.offset = fd; 
        block.dirty = true;

        cacheList.push_front(block);
        if (cacheList.size() > capacity) {
            cacheList.pop_back();
        }

        return bytesWritten;
    }

    off_t lab2_lseek(intptr_t fd, off_t offset, int whence) {
        lock_guard<mutex> lock(cacheMutex);

        if (fileHandles.find(fd) == fileHandles.end()) {
            return -1; 
        }

        HANDLE fileHandle = fileHandles[fd];
        LARGE_INTEGER distance;
        distance.QuadPart = offset;
        LARGE_INTEGER newOffset;

        DWORD moveMethod;
        switch (whence) {
        case SEEK_SET:
            moveMethod = FILE_BEGIN;
            break;
        case SEEK_CUR:
            moveMethod = FILE_CURRENT;
            break;
        case SEEK_END:
            moveMethod = FILE_END;
            break;
        default:
            return -1; 
        }

        if (!SetFilePointerEx(fileHandle, distance, &newOffset, moveMethod)) {
            return -1;
        }

        return static_cast<off_t>(newOffset.QuadPart);
    }

    int lab2_fsync(intptr_t fd) {
        lock_guard<mutex> lock(cacheMutex);

        if (fileHandles.find(fd) == fileHandles.end()) {
            return -1; 
        }

        HANDLE fileHandle = fileHandles[fd];
        for (auto it = cacheList.begin(); it != cacheList.end(); ++it) {
            if (it->offset == fd && it->dirty) {
                DWORD bytesWritten;
                if (!WriteFile(fileHandle, it->data.data(), static_cast<DWORD>(it->data.size()), &bytesWritten, NULL)) {
                    return -1; 
                }
                it->dirty = false;
            }
        }
        return 0;
    }
};
```
![benchmark_results1](https://github.com/user-attachments/assets/5ec0d035-a2bc-485a-a164-abe8fc3b8b9b)
![benchmark_results](https://github.com/user-attachments/assets/cc072994-9f5c-44a8-bb28-fbb2d9d73322)

Использование кеша MRU может ускорить выполнение программы, если данные имеют локальность по времени или часто повторяются. Однако, если данные распределены равномерно и нет повторяющихся элементов, то кеш MRU может не дать значительного прироста производительности. В моём случае, влияние кеша будет минимальным, потому что данные достаточно уникальны и распределены равномерно.
