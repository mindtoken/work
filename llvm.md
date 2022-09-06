Build
cmake -DCMAKE_BUILD_TYPE=Release  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;compiler-rt;libcxx;libcxxabi;lld"  -G "Ninja" -DCMAKE_INSTALL_PREFIX=/home/yuanpeng/usr -DLLVM_TARGETS_TO_BUILD=X86 ../llvm
cmake --build .
cmake --build . --target install

cmake -G Ninja ../llvm \
   -DLLVM_ENABLE_PROJECTS=mlir \
   -DLLVM_BUILD_EXAMPLES=ON \
   -DLLVM_TARGETS_TO_BUILD="X86;NVPTX;AMDGPU" \
   -DCMAKE_BUILD_TYPE=Release \
   -DLLVM_ENABLE_ASSERTIONS=ON
https://mlir.llvm.org/getting_started/
