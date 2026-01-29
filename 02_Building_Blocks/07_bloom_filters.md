# Bloom Filters

## Overview

A Bloom filter is a space-efficient probabilistic data structure that tests whether an element is a member of a set.

```
Key insight:
- "Definitely NOT in set" → Always correct
- "Probably in set" → May be wrong (false positive)

No false negatives, possible false positives
```

## How It Works

### Structure

```
Bit array of size m:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9

k hash functions: h1, h2, h3, ...
```

### Adding an Element

```
Add "apple":
h1("apple") = 2
h2("apple") = 5
h3("apple") = 8

Set bits at positions 2, 5, 8:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9
```

### Querying

```
Check "apple":
h1("apple") = 2 → bit[2] = 1 ✓
h2("apple") = 5 → bit[5] = 1 ✓
h3("apple") = 8 → bit[8] = 1 ✓
All bits set → "Probably in set"

Check "banana":
h1("banana") = 3 → bit[3] = 0 ✗
→ "Definitely NOT in set" (no need to check other hashes)
```

### False Positives

```
Add "apple" and "orange":
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 1 │ 0 │ 1 │ 1 │ 0 │ 0 │ 1 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

Check "grape" (never added):
h1("grape") = 2 → bit[2] = 1 ✓
h2("grape") = 4 → bit[4] = 1 ✓
h3("grape") = 1 → bit[1] = 1 ✓
All bits set → "Probably in set" (FALSE POSITIVE!)

Bits were set by other elements
```

## Parameters

### Optimal Configuration

```
n = expected number of elements
p = desired false positive probability
m = bit array size
k = number of hash functions

Formulas:
m = -n × ln(p) / (ln(2))²
k = (m/n) × ln(2)

Example: n=1,000,000, p=1%
m = 9,585,059 bits ≈ 1.14 MB
k = 7 hash functions
```

### Trade-offs

| Parameter | Effect |
|-----------|--------|
| Larger m | Lower false positive rate, more memory |
| More k | Lower false positive rate (to a point), slower |
| More elements | Higher false positive rate |

## Properties

| Property | Value |
|----------|-------|
| False negatives | Never |
| False positives | Possible (configurable rate) |
| Deletion | Not supported (standard Bloom filter) |
| Space | O(m) bits |
| Insert time | O(k) hash computations |
| Query time | O(k) hash computations |

## Use Cases

### Database Query Optimization

```
Before disk lookup, check Bloom filter:

Query: SELECT * FROM users WHERE id = 123

┌─────────────────────────────────────────────────────┐
│  Check Bloom filter first                           │
│                                                     │
│  "Is 123 in this SSTable?"                         │
│      │                                              │
│      ├── NO → Skip SSTable (avoid disk read)       │
│      │                                              │
│      └── MAYBE → Read from disk to confirm         │
└─────────────────────────────────────────────────────┘

Used by: LevelDB, RocksDB, Cassandra, HBase
```

### Cache Filtering

```
Before checking cache:

Request for key X
     │
     ▼
┌─────────────────┐
│  Bloom Filter   │
│  "Is X cached?" │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    NO       MAYBE
    │         │
    ▼         ▼
 Fetch     Check cache
 from DB   (may be false positive)
```

### Malicious URL Detection

```
Browser checks URL against Bloom filter:

User visits url.com
         │
         ▼
┌─────────────────────┐
│ Local Bloom Filter  │
│ (known bad URLs)    │
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    │             │
    NO           MAYBE
    │             │
    ▼             ▼
  Allow       Check with
  access      remote service

Google Safe Browsing uses this approach
```

### Spell Checking

```
Dictionary Bloom filter:

Check "teh":
- Bloom filter says NO → Definitely misspelled
- Bloom filter says MAYBE → Might be correct word
```

### Network Systems

```
- CDN: "Is this content cached at this edge?"
- Distributed systems: "Does this node have this data?"
- Routers: "Is this IP in the blocklist?"
```

## Variants

### Counting Bloom Filter

Supports deletion by using counters instead of bits.

```
Standard:  [0, 0, 1, 0, 1, 0]  (can't delete)
Counting:  [0, 0, 2, 0, 1, 0]  (counter per position)

Add x (positions 2, 4): [0, 0, 3, 0, 2, 0]
Delete x: [0, 0, 2, 0, 1, 0]

Trade-off: Uses more space (counters vs bits)
```

### Cuckoo Filter

Better than Bloom for many use cases.

```
Supports:
- Efficient deletion
- Better space efficiency
- Faster lookups

Uses cuckoo hashing with fingerprints
```

### Scalable Bloom Filter

Grows as elements are added.

```
When filter is "full", add new filter:

┌─────────────────┐
│ Bloom Filter 1  │ (first million elements)
└─────────────────┘
         │
┌─────────────────┐
│ Bloom Filter 2  │ (next million)
└─────────────────┘
         │
┌─────────────────┐
│ Bloom Filter 3  │ (and so on)
└─────────────────┘

Query checks all filters
```

## Implementation

### Simple Python Example

```python
import hashlib

class BloomFilter:
    def __init__(self, size, num_hashes):
        self.size = size
        self.num_hashes = num_hashes
        self.bit_array = [0] * size

    def _hashes(self, item):
        """Generate k hash values for item."""
        hashes = []
        for i in range(self.num_hashes):
            h = hashlib.md5(f"{item}{i}".encode()).hexdigest()
            hashes.append(int(h, 16) % self.size)
        return hashes

    def add(self, item):
        """Add item to filter."""
        for pos in self._hashes(item):
            self.bit_array[pos] = 1

    def contains(self, item):
        """Check if item might be in filter."""
        return all(self.bit_array[pos] for pos in self._hashes(item))

# Usage
bf = BloomFilter(size=1000, num_hashes=3)
bf.add("apple")
bf.add("orange")

print(bf.contains("apple"))   # True (correct)
print(bf.contains("banana"))  # False (correct)
print(bf.contains("grape"))   # Might be True (false positive possible)
```

## Real-World Examples

| System | Use Case |
|--------|----------|
| Cassandra | SSTable lookup optimization |
| LevelDB/RocksDB | Skip unnecessary disk reads |
| Ethereum | Transaction verification |
| Chrome | Safe browsing URL check |
| Akamai | CDN cache checking |
| Medium | Recommend articles not seen |
| Bitcoin | SPV wallet transaction filtering |

## Comparison with Other Structures

| Structure | Space | False Positives | Deletion | Use Case |
|-----------|-------|-----------------|----------|----------|
| Hash Set | O(n) | None | Yes | Exact membership |
| Bloom Filter | O(m) fixed | Yes | No | Probabilistic membership |
| Cuckoo Filter | O(n) | Yes (lower) | Yes | Better Bloom alternative |
| HyperLogLog | O(log log n) | N/A | N/A | Cardinality estimation |

## Interview Talking Points

1. "Bloom filters trade accuracy for space efficiency"
2. "No false negatives - if it says NO, it's definitely NO"
3. "Used to avoid expensive operations like disk reads"
4. "Can't delete from standard Bloom filter"
5. "Size and false positive rate are configurable trade-offs"

## Common Interview Questions

1. **Q: What is a Bloom filter and when would you use it?**
   A: Space-efficient probabilistic set membership test. Use when you want to quickly eliminate non-members before expensive operations.

2. **Q: Can you delete from a Bloom filter?**
   A: Not from standard Bloom filter (might delete bits needed by other elements). Counting Bloom filter supports deletion.

3. **Q: How do you choose the right size?**
   A: Based on expected elements (n) and acceptable false positive rate (p). Formula: m = -n × ln(p) / (ln(2))².

4. **Q: Give an example use case.**
   A: Database checks Bloom filter before disk read. If filter says "not present," skip disk access. False positives just mean unnecessary disk reads.

## Key Takeaways

- Bloom filters are space-efficient for membership testing
- "Not in set" is always correct
- "In set" might be wrong (false positive)
- Can't delete from standard Bloom filter
- Great for filtering before expensive operations

## Further Reading

- "Bloom Filters by Example" (online resource)
- Original paper by Burton Bloom (1970)
- Cuckoo Filter paper
- LSM-tree and Bloom filter integration
