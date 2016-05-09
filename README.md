# MUM Hash
* MUM hash is a **fast non-cryptographic hash function**
  suitable for different hash table implementations
* MUM means **MU**ltiply and **M**ix
  * It is a name of the base transformation on which hashing is implemented
  * Modern processors have a fast logic to do long number multiplications
  * It is very attractable to use it for fast hashing
    * For example, 64x64-bit multiplication can do the same work as 32
      shifts and additions
  * I'd like to call it Multiply and Reduce.  Unfortunately, MUR
    (MUltiply and Rotate) is already taken for famous hashing
    technique designed by Austin Appleby
  * I've chosen the name also as I am releasing it on Mother's day
* MUM hash passes **all** [SMHasher](https://github.com/aappleby/smhasher) tests
  * For comparison, only 4 out of 15 non-cryptographic hash functions
    in SMHasher passes the tests, e.g. well known FNV, Murmur2,
    Lookup, and Superfast hashes fail the tests
* MUM algorithm is **simpler** than City64 and Spooky ones
* MUM is specifically **designed for 64-bit CPUs** (Sorry, I did not want to
  spend my time on dying architectures)
  * Still MUM will work for 32-bit CPUs and it will be sometimes
    faster Spooky and City
* On x86-64 MUM hash is **faster** than City64 and Spooky on all tests except for one
  test for the bulky speed
  * Starting with 240-byte strings, City uses Intel SSE4.2 crc32 instruction
  * I could use the same instruction but I don't want to complicate the algorithm
  * In typical scenario, such long strings are rare.  Usually another
    interface (see `mum_hash_step`) is used for hashing big data structures
* MUM has a **fast startup**.  It is particular good to hash small keys
  which are a majority of hash table applications

# MUM implementation details

* Input 64-bit data are randomized by 64x64->128 bit multiplication and mixing
  high- and low-parts of the multiplication result by using an addition.
  The result is mixed with the current state by using XOR
  * Instead of the addition for mixing high- and low- parts, XOR could be
    used
    * Using the addition instead of XOR improves performance by about
      10% on Haswell and Power7
* Prime numbers randomly generated with the equal probability of their
  bit values are used for the multiplication
* When all primes are used once, the state is randomized and the same
  prime numbers are used again for subsequent data randomization
* Major loop is transformed to be **unrolled** by compiler to benefit from the
  compiler instruction scheduling optimization and OOO instruction
  execution in modern CPUs
* x86-64 *AVX2* insn `MULX` is used on processors where it is implemented
  * Using `MULX` increases the bulk speed upto 25%
  * Although on modern Intel processors `MULQ` takes 3-cylces vs. 4 for
    `MULX`, `MULX` permits more freedom in insn scheduling as it uses less
    fixed registers.  That is a reason for the improvement
  * To make the code portable, **GCC function multiversioning** is used.
    The best code is chosen depending on the used processor features
* AARCH64 128-bit result multiplication is very slow as it is
  implemented by a GCC library function
  * To use only 2 insns for such multiplication one GCC **asm extension**
    was added
    
   
# MUM benchmarking vs Spooky, City64, and SipHash24

* Here are the results of benchmarking MUM and the fastest
  non-cryptographic hash functions I know:
  * Google City64 (sources are taken from SMHasher)
  * Bob Jenkins Spooky (sources are taken from SMHasher)
  * Yann Collet's xxHash64 (sources are taken from the
    [original repository](https://github.com/Cyan4973/xxHash))
* Murmur hash functions are slower so I don't compare it here
* I also added J. Aumasson and D. Bernstein's
  [SipHash24](https://github.com/veorq/SipHash) for the comparison as it
  is a popular choice for hash table implementation these days
* Measurements were done on 3 different architecture machines:
  * 4.2 GHz Intel i7-4790K
  * 3.55 GHz Power7
  * APM X-Gene Mustang Board
* Each test was run 3 times and the minimal time was taken
  * GCC-4.8 was used for PPC and AARCH64 machines and GCC-4.9 was used for Intel one
  * `-O3` was used for all compilations
  * The strings were generated by `rand` calls
  * The strings were aligned to see a hashing speed better
  * No constant propagation for string length is forced.  Otherwise,
    the results for MUM hash would be even better
  
# Intel i7-4790K (4.2GHz)

|                           |  MUM | City64|  Spooky| xxHash64|SipHash24| 
:---------------------------|-----:|------:|-------:|--------:|--------:|
8 bytes  (1,280M strings)   | 5.52s| 10.83s|  10.35s| 16.22s  |   26.43s|
16 bytes (1,280M strings)   | 7.38s| 11.15s|  18.29s| 18.07s  |   31.11s|
32 bytes (1,280M strings)   | 8.06s| 13.15s|  18.50s| 19.06s  |   42.73s|
64 bytes (1,280M strings)   |11.59s| 13.75s|  26.59s| 25.74s  |   63.38s|
128 bytes (1,280M strings)  |17.16s| 18.99s|  42.88s| 18.99s  |  106.77s|
1KB (100M strings)          | 6.61s|  6.37s|   8.80s|  8.45s  |   53.20s|

# Power7 (3.55GHz)

|                           | MUM  | City64|  Spooky| xxHash64|SipHash24|
:---------------------------|-----:|------:|-------:|--------:|--------:|
8 bytes  (1,280M strings)   | 8.63s| 26.61s|  27.04s| 77.30s  |   55.64s|
16 bytes (1,280M strings)   |18.30s| 26.07s|  41.77s| 79.51s  |   62.53s|
32 bytes (1,280M strings)   |19.29s| 30.03s|  41.01s| 76.90s  |   84.89s|
64 bytes (1,280M strings)   |21.95s| 33.71s|  57.68s| 82.59s  |  122.49s|
128 bytes (1,280M strings)  |30.20s| 57.68s|  90.56s| 96.09s  |  197.45s|
1KB (100M strings)          |14.31s| 16.93s|  16.26s| 20.96s  |   96.78s|

# AARCH64 (APM X-Gene)

|                           | MUM  | City64|  Spooky| xxHash64|SipHash24|
:---------------------------|-----:|------:|-------:|--------:|--------:|
8 bytes  (1,280M strings)   |14.43s| 27.27s|  27.80s| 56.69s  |   68.44s|
16 bytes (1,280M strings)   |22.94s| 28.87s|  43.85s| 62.57s  |   78.08s|
32 bytes (1,280M strings)   |29.94s| 33.69s|  47.06s| 79.14s  |   98.93s|
64 bytes (1,280M strings)   |43.83s| 40.64s|  64.70s| 90.38s  |  141.71s|
128 bytes (1,280M strings)  |70.05s| 74.33s|  98.93s|112.83s  |  235.83s|
1KB (100M strings)          |34.05s| 22.91s|  24.23s| 33.42s  |  104.98s|

# Vectorization
* A major loop in function `_mum_hash_aligned` could be vectorized
  using vector multiplication, addition, and shuffle instructions
* Unfortunately, x86-64 CPUs currently does not have vector
  multiplication `64 x 64-bit -> 128-bit`
* AVX2 CPUs only have vector multiplication `32 x 32-bit -> 64-bit`
  * One such vector instruction makes 4 multiplication which is
    roughly equivalent what one `MULQ/MULX` insn does but it has bigger
    latency time than `MULQ/MULX`
  * Therefore all my vectorized code I tried was slower
* If Intel introduces a new vector insn for `64 x 64-bit -> 128-bit`
  multiplication, potentially it could increase MUM speed up to 2
  times (may be less as it requires to provide aligned data)

# Using cryptographic vs. non-cryptographic hash function
  * People worrying about denial attacks based on generating hash
    collisions started to use cryptographic hash functions in hash tables
  * Cryptographic functions are very slow
    * *sha1* is about 20-30 slower than MUM and City on the bulk speed tests
    * The new fastest cryptographic hash function *SipHash* is up to 10
      times slower
  * MUM is also *resistant* to preimage attack (finding a string with given hash)
    * To make hard moving to previous state values we use mostly 1-to-1 one way
      function `lo(x*C) + hi(x*C)` where C is a constant.  Brute force
      solution of equation `f(x) = a` probably requires `2^63` tries.
      Another used function equation `x ^ y = a` has a `2^64`
      solutions.  It complicates finding the overal solution further
  * If somebody is not convinced, you can use **randomly chosen
    multiplication constants** (see function `mum_hash_randomize`).
    Finding a string with a given hash even if you know a string with such
    hash probably will be close to finding two or more solutions of
    *Diophantine* equations
  * If somebody is still not convinced, you can implement hash tables
    to **recognize the attack and rebuild** the table using MUM function
    with the new multiplication constants
  * Analogous approach can be used if you use weak hash function as
    MurMur or City.  Instead of using cryptographic hash functions
    **all the time**, hash tables can be implemented to recognize the
    attack and rebuild the table and start using a cryptographic hash
    function
  * This approach solves the speed problem and permits to switch easily to a new
    cryptographic hash function if a flaw is found in the old one, e.g. switching from
    SipHash to SHA2
  
# How to use MUM
* Please just include file `mum.h` into your C/C++ program and use the following functions:
  * optional `mum_hash_randomize` for choosing multiplication constants randomly
  * `mum_hash_init`, `mum_hash_step`, and `mum_hash_finish` for hashing complex data structures
  * `mum_hash64` for hashing a 64-bit data
  * `mum_hash` for hashing any continuous block of data
* To compare MUM speed with Spooky, City64, and SipHash24 on your machine go to
  the directory `src` and run a script

```
sh bench
```

* The script will compile source files and run the tests printing the
  results

# Crypto-hash function MUM512
  * MUM is not designed to be a crypto-hash
    * The key (seed) and state are only 64-bit which are not crypto-level ones
    * The result can be different for different targets (BE/LE
      machines, 32- and 64-bit machines) as for other hash functions, e.g. City (hash can be
      different on SSE4.2 nad non SSE4.2 targets) or Spooky (BE/LE machines)
      * If you need the same MUM hash independent on the target, please
        define macro `MUM_TARGET_INDEPENDENT_HASH`
  * There is a variant of MUM called MUM512 which can be a **candidate**
    for a crypto-hash function and keyed crypto-hash function and
    might be interesting for researchers
    * The **key** is **256**-bit
    * The **state** and the **output** are **512**-bit
    * The **block** size is **512**-bit
    * It uses 128x128->256-bit multiplication which is analogous to about
      64 shifts and additions for 128-bit block word instead of 80
      rounds of shifts, additions, logical operations for 512-bit block
      in sha2-512.
  * It is **only a candidate** for a crypto hash function
    * I did not make any differential crypto-analysis or investigated
      probabilities of different attacks on the hash function (sorry, it
      is too big job)
      * I might be do this in the future as I am interesting in
        differential characteristics of the MUM512 base transformation
        step (128x128-bit multiplications with addition of high and
        low 128-bit parts)
      * I am interesting also in the right choice of the multiplication constants
      * May be somebody will do the analysis.  I will be glad to hear anything.
        Who knows, may be it can be easily broken as Nimbus cipher.
    * The current code might be also vulnerable to timing attack on
      systems with varying multiplication instruction latency time.
      There is no code for now to prevent it
  * To compare the MUM512 speed with the speed of SHA-2 (SHA512) and
    SHA-3 (SHA3-512) go to the directory `src` and run a script `sh bench-crypto`
    * SHA-2 and SHA-3 code is taken from [RHash](https://github.com/rhash/RHash.git)
  * Here is the speed of the crypto hash functions on 4.2 GHz Intel i7-4790K:

|                        | MUM512 | SHA2  |  SHA3  |
:------------------------|-------:|------:|-------:|
10 bytes (20 M texts)    | 0.73s  | 0.65s |  1.13s |
100 bytes (20 M texts)   | 1.08s  | 0.65s |  2.21s |
1000 bytes (20 M texts)  | 4.01s  | 4.82s |  15.3s |
10000 bytes (5 M texts)  | 7.86s  |11.74s |  37.3s |

# Pseudo-random generators
  * Files `mum-prgn.h` and `mum512-prng.h` provides pseudo-random
    functions based on MUM and MUM512 hash functions
  * All PRNGs pass *NIST Statistical Test Suite for Random and
    Pseudorandom Number Generators for Cryptographic Applications*
    (version 2.2.1) with 1000 bitstreams each containing 1M bits
    * Although MUM PRNG pass the test, it is not a cryptographically
      secure PRNG as the hash function used for it
  * To compare the PRNG speeds go to
    the directory `src` and run a script `sh bench-prng`
  * For the comparison I wrote crypto-secured Blum Blum Shub PRNG
    (file `bbs-prng.h`) and PRNGs based on fast cryto-level hash
    functions in ChaCha stream cipher (file `chacha-prng.h`) and
    SipHash24 (file `sip24-prng.h`).
    * The additional PRNGs also pass the Statistical Test Suite
  * For the comparison I also added the fastest PRNG
    [xoroshiro128+](http://xoroshiro.di.unimi.it/xoroshiro128plus.c)
    (it is not a crypto level PRNG)
  * Here is the speed of the PRNGs in millions generated PRNs
    per second on 4.2 GHz Intel i7-4790K:

|                        | M prns/sec  |
:------------------------|------------:|
BBS                      | 0.057       |
ChaCha                   | 106         |
SipHash24                | 383         |
MUM512                   |  67         |
MUM                      | 687         |
XOROSHIRO128+            |1145         |
GLIBC RAND               | 169         |

