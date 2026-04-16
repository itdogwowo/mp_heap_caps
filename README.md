# mp_heap_caps

MicroPython user C module that exposes ESP-IDF `heap_caps_*` allocators and heap statistics to Python.

- Module name in MicroPython: `heap_caps`
- Primary use cases: DMA-capable buffers, PSRAM (SPIRAM) buffers, and querying available heap by capability.

## Features

- Allocate/free/realloc with a capability bitmask (`caps`)
- Optional aligned allocation (ESP32 only)
- Heap statistics (ESP32 only)
- Exposes common `MALLOC_CAP_*` constants to Python

## API

### Allocation

All allocation functions return a writable `memoryview` on success, or `None` on failure.

- `heap_caps.malloc(size, caps) -> memoryview | None`
  - Allocates `size` bytes in a heap region matching `caps`
  - Current implementation uses `heap_caps_calloc(1, size, caps)` (buffer is zero-initialized)
- `heap_caps.free(buf) -> None`
  - Frees a buffer previously returned by this module
  - `buf` may be a single `memoryview` or a list of `memoryview` objects (each element must be a separately allocated buffer)
- `heap_caps.realloc(buf, size, caps) -> memoryview | None`
  - Reallocates an existing buffer into a heap region matching `caps`
- `heap_caps.calloc(count, size, caps) -> memoryview | None`
  - Allocates `count * size` bytes, zero-initialized
- `heap_caps.aligned_alloc(alignment, size, caps) -> memoryview | None` (ESP32 only)
- `heap_caps.aligned_calloc(alignment, count, size, caps) -> memoryview | None` (ESP32 only)

### Heap statistics (ESP32 only)

- `heap_caps.get_free_size(caps) -> int`
- `heap_caps.get_total_size(caps) -> int`
- `heap_caps.get_largest_free_block(caps) -> int`

### Capability constants

The module exposes common ESP-IDF constants, for example:

- `heap_caps.CAP_DMA`
- `heap_caps.CAP_INTERNAL`
- `heap_caps.CAP_SPIRAM`
- plus other `CAP_*` values available on the target.

## Examples

### Print DMA / PSRAM / Internal heap stats

```python
import heap_caps

def show(title, caps):
    print(title)
    print("  free   :", heap_caps.get_free_size(caps))
    print("  total  :", heap_caps.get_total_size(caps))
    print("  largest:", heap_caps.get_largest_free_block(caps))
    print()

show("DMA (CAP_DMA)", heap_caps.CAP_DMA)
show("SPIRAM (CAP_SPIRAM)", heap_caps.CAP_SPIRAM)
show("INTERNAL (CAP_INTERNAL)", heap_caps.CAP_INTERNAL)
```

You can combine requirements using bitwise OR:

```python
show("INTERNAL|DMA", heap_caps.CAP_INTERNAL | heap_caps.CAP_DMA)
```

### Allocate a DMA-capable buffer

```python
import heap_caps

buf = heap_caps.malloc(32 * 1024, heap_caps.CAP_DMA)
if buf is None:
    raise MemoryError("no DMA heap")

# Use buf with your driver...

heap_caps.free(buf)
```

### SPI LCD pattern (PSRAM framebuffer + DMA bounce buffer)

Typical approach on ESP32: keep large images/framebuffers in PSRAM, and use a smaller internal DMA buffer for SPI transfers.

```python
import heap_caps

frame = heap_caps.malloc(240 * 320 * 2, heap_caps.CAP_SPIRAM)  # RGB565 framebuffer in PSRAM
tx = heap_caps.malloc(32 * 1024, heap_caps.CAP_DMA)           # DMA bounce buffer (internal)

if frame is None or tx is None:
    raise MemoryError

# Fill `frame` (PSRAM), then copy chunks into `tx` and send via SPI.
# Example pseudo flow:
# for each chunk:
#     tx[:n] = frame[offset:offset+n]
#     spi.write(tx[:n])

heap_caps.free(tx)
heap_caps.free(frame)
```

## Safety notes (important)

- You must call `heap_caps.free()` for buffers allocated by this module. They are not managed by MicroPython GC.
- Only free the original `memoryview` returned by `heap_caps.*alloc*`/`realloc`.
  - Do not pass a slice (e.g. `buf[16:]`) to `heap_caps.free()`. A slice points to an offset address and freeing it is invalid.
  - Do not pass a `bytearray`/`memoryview` that was not allocated by this module.
- `get_largest_free_block(caps)` is usually the most useful number for DMA, because DMA often requires a single contiguous block.

## Build / Integration

### CMake-based ports (e.g. ESP32 / ESP-IDF)

Add this repository as a user C module and include its [micropython.cmake](file:///c:/Users/bl91920/Documents/code/git/mp_heap_caps/micropython.cmake).

This module selects:

- `esp32_src/heap_caps.c` when `ESP_PLATFORM` is set
- `common_src/heap_caps.c` otherwise

### Make-based ports

Include [micropython.mk](file:///c:/Users/bl91920/Documents/code/git/mp_heap_caps/micropython.mk) as a user C module.

## Chinese README

- See [README.zh-TW.md](file:///c:/Users/bl91920/Documents/code/git/mp_heap_caps/README.zh-TW.md)
