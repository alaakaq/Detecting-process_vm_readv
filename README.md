# Detecting process_vm_readv and process_vm_writev

Disclaimer: This information is shared for educational purposes only.

## Description
To make the long story short, process_vm_readv, process_vm_writev functions are used in rooted devices and virtual enviroments to access memory of another process (Read\Write), such as reading players data and drawing ESP (Boxes, Tracelines, etc.).

## How to detect
Android kernel neither notifies your process nor exposes a direct API to detect such attacks, fortunately there is a workaround to do so. 
you can take advantage of how kernel manages memory pages, if a memory was created using mmap, the memory won't be backed by physical memory page, unless it was accessed either internally or externally, then it will be backed immediately by kernel, however in both cases it will still show up in virtually mapped regions (e.g. /proc/self/maps).

## Notes
According to man (Linux manual page)

For non file backed memory regions:
> MAP_ANONYMOUS
              The mapping is not backed by any file; its contents are
              initialized to zero.

For file backed memory regions:

> The contents of a file mapping (as opposed to an anonymous
       mapping; see MAP_ANONYMOUS below), are initialized using length
       bytes starting at offset offset in the file (or other object)
       referred to by the file descriptor fd

## Magic
- ### Imagine the following game structure
```cpp
struct Players {
    ...
}

struct Engine {
    ...
    Players** pPlayers;
    ...
}

Engine* gpEngine;
```

- ### Introduce a trap
You have to create a trap memory region using mmap
```cpp
gTrapMem = mmap(nullptr, PAGE_SIZE, PROT_READ, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
```

- ### Assign Player list object to the trap region
Normally it should look like this
```cpp
static Players* intermediatePointer = new Players();
gpEngine->pPlayers = &intermediatePointer;
```
In our case
```cpp
static Players* intermediatePointer = new Players();
gpEngine->pPlayers = reinterpret_cast<Player**>(gTrapMem);
```

and attacker will try to check pointer like this

```cpp
if (gpEngine->pPlayers != nullptr) {
    if (*gpEngine->pPlayers != nullptr) { // Here attacker has accessed our trap memory instead of real one
        ...
    }
}
```

Thus now attacker has left a trace for us to detect, let's see how.

- ### Detect
As I said, kernel will only peg/back virtual memory by physical memory if it was accessed, otherwise no.

Our helper function:
```cpp
bool isPageBackedByPhysicalMemory(void *addr, uint64_t &entry) {
    int fd = open("/proc/self/pagemap", O_RDONLY);
    if (fd == -1) {
        LOGE("Failed to open /proc/self/pagemap");
        return false;
    }

    off_t offset = (reinterpret_cast<unsigned long>(addr) / PAGE_SIZE) * sizeof(entry);
    if (pread64(fd, &entry, sizeof(entry), offset) != sizeof(entry)) {
        LOGE("Failed to read from /proc/self/pagemap");
        close(fd);
        return false;
    }

    close(fd);

    bool isPagePresent = entry & (1ULL << 63);
    return isPagePresent;
}
```

Usage:
```cpp
uint64_t entry;
bool isPagePresent = isPageBackedByPhysicalMemory(gTrapMem, entry);
if (isPagePresent) {
    // BOOM: attack detected, the memory was accessed.
    // TODO: Take action...
}
```
