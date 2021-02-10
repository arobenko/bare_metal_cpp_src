## Test Applications

The [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project contains 
several simple test application, which are intended to be used for binary code 
analysis only and not to be executed on the target platform. This applications 
reside in [src/test_cpp](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp) 
directory. In order to properly analyse the code that compiler produces for production environment, let's compile all the applications in Release mode:
```
> git clone https://github.com/arobenko/embxx_on_rpi.git
> mkdir -p <build_dir_somewhere>
> cd <build_dir_somewhere>
> cmake -DCMAKE_BUILD_TYPE=Release <path/to/embxx_on_rpi>
> VERBOSE=1 make
```

The listing file of every application will be `<build_dir_somewhere>/src/test_cpp/<app_name>/kernel.list`. 

