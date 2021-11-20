
### Hello World Example using ARM Cross Compiler


```bash
# Pull and Run a Docker Image with ARM GCC.
$ cd examples/helloworld_cmake
$ docker pull kubasejdak/arm-linux-gnueabihf-gcc
$ docker run -it -v $(pwd)/:/helloworld kubasejdak/arm-linux-gnueabihf-gcc
# Now running inside the Container.
$ cd /helloworld
$ make arm
$ make
$ ls -sh build
total 40K
 16K CMakeCache.txt     0 CMakeFiles  8.0K Makefile     0 cmake_install.cmake   16K helloworld
# Clean and build a Linux version for x86_64.
$ make clean linux build
$ ls -sh build
total 40K
 16K CMakeCache.txt     0 CMakeFiles  8.0K Makefile     0 cmake_install.cmake   16K helloworld
```
