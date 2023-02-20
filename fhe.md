# INSTALL
## SEAL
```
# Disable CXX17: error: no member named 'aligned_alloc' in the global namespace
$ cmake -S . -B build -DSEAL_USE_ALIGNED_ALLOC=OFF -DSEAL_USE_CXX17=OFF -DCMAKE_INSTALL_PREFIX=/Users/mac/cloud/fhe/usr
$ cmake --build build
$ cmake --install build

# Examples, Tests, and Benchmarks
cd native/<examples|tests|bench>
cmake -S . -B build -DSEAL_ROOT=/Users/mac/cloud/fhe/usr
cmake --build build

```
Performance:
$ cd seal/main/native/examples
$ ./build/bin/sealexamples

Build SEAL with Intel HEXL

## TenSEAL
