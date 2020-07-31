# 编译参数和区别

## 目标平台

Clang targets with ARM
When building for ARM, Clang changes the target based on the presence of the -march=armv7-a and/or -mthumb compiler flags:

Table 1. Specifiable -march values and their resulting targets.

| -march value | Resulting target | 
| ------------ | ------------ |
| -march=armv7-a |	armv7-none-linux-androideabi | 
| -mthumb |	thumb-none-linux-androideabi | 
| Both -march=armv7-a and -mthumb | thumbv7-none-linux-androideabi | 

You may also override with your own -target if you wish.

clang and clang++ should be drop-in replacements for gcc and g++ in a makefile. When in doubt, use the following options when invoking the compiler to verify that they are working properly:

- -v to dump commands associated with compiler driver issues
- -### to dump command line options, including implicitly predefined ones.
- -x c < /dev/null -dM -E to dump predefined preprocessor definitions
- -save-temps to compare *.i or *.ii preprocessed files.
ABI Compatibility

