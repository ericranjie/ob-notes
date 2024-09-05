```cpp
//
class FixedAllocator {
 private:
  std::size_t blocksize;
  unsigned char nblocks;
  typedef std::vector<Chunk> Chunks;
  Chunks chunks;
  Chunk* alloc_chunk;
  Chunk* dealloc_chunk;
}
```