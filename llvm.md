# MLIR
## Build on MAC M1
```
# install ninja: brew install ninja
% pwd
/path/llvm-project-llvmorg-16.0.0/build
% cmake -G Ninja ../llvm -DLLVM_ENABLE_PROJECTS=mlir -DLLVM_BUILD_EXAMPLES=ON -DLLVM_TARGETS_TO_BUILD="Native" -DCMAKE_BUILD_TYPE=Release  -DLLVM_ENABLE_ASSERTIONS=ON -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
% cmake --build . --target check-mlir | tee log.build_mlir
```

## Toy:
```
$ cd seal/main/native/examples
$ ./build/bin/sealexamples
```
