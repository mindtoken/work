# INSTALL
## SEAL
```
# Disable CXX17: error: no member named 'aligned_alloc' in the global namespace
$ cmake -S . -B build -DSEAL_USE_ALIGNED_ALLOC=OFF -DSEAL_USE_CXX17=OFF -DCMAKE_INSTALL_PREFIX=/Users/mac/cloud/fhe/usr
$ cmake --build build
```
## TenSEAL
