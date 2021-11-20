

default: build

linux: clean
	mkdir build
	cd build; cmake ..

arm: clean
	mkdir build
	cd build; cmake -DCMAKE_TOOLCHAIN_FILE=arm-toolchain.cmake ..

build:
	cd build; make

clean:
	rm -rf build

.PHONY: linux arm build clean
