# SEAL
## Build
```
# Disable CXX17: error: no member named 'aligned_alloc' in the global namespace
$ cmake -S . -B build -DSEAL_USE_ALIGNED_ALLOC=OFF -DSEAL_USE_CXX17=OFF -DCMAKE_INSTALL_PREFIX=/Users/mac/cloud/fhe/usr
$ cmake --build build
$ cmake --install build

# Examples, Tests, and Benchmarks
cd native/<examples|tests|bench>
cmake -S . -B build -DSEAL_ROOT=/Users/mac/cloud/fhe/usr
cmake --build build

#Build SEAL with Intel HEXL
```

## Performance:
```
$ cd seal/main/native/examples
$ ./build/bin/sealexamples
```

## Test
```
# Refer CMakeFile in native/examples
$ cmake -S . -B build -DSEAL_ROOT=/Users/mac/cloud/fhe/usr
$ cmake --build build
```

# openfhe
## Build
https://openfhe-development.readthedocs.io/en/latest/sphinx_rsts/intro/installation/macos.html
```
// Install OpenMP Library: 
$ brew install libomp

// Install OpenFHE
$ git clone git@github.com:openfheorg/openfhe-development.git main
$ mkdir build
$ cd build 
$ cmake -DCMAKE_CROSSCOMPILING=1 -DRUN_HAVE_STD_REGEX=0 -DRUN_HAVE_POSIX_REGEX=0 -DCMAKE_INSTALL_PREFIX=/Users/yuanpeng/compiler/usr ../main
$ make -j8
$ make install
$ ./bin/examples/pke/simple-real-numbers
```
## Build App
https://openfhe-development.readthedocs.io/en/latest/sphinx_rsts/intro/building_user_applications.html
```
Use CMakeLists.User.txt
```

