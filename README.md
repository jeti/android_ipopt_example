This is perhaps the simplest possible example project that one can make demonstrating how to use IPOPT in Android. To accomplish this, you will need the precompiled IPOPT libraries for Android that are available in the [release section](https://github.com/jeti/android_ipopt/releases) of this repo:

- https://github.com/jeti/android_ipopt

or you can build the libraries yourself with the script provided there. Up to you...

There are just a few steps necessary to get IPOPT working in an Android project, as demonstrated in [this video](https://youtu.be/7hNbvoRIvZE). Here is a recap of what we did to create this repo (you can follow these steps to create your own app from scratch):

1. Create a new Android project with C++ support. This project should have the structure 

   ```
   project/
   ---app/
   ------build.gradle
   ------CMakeLists.txt
   ---------libs/
   ---------src/
   ------------main/
   ------------cpp/   
   ```

2. Download the shared libraries from https://github.com/jeti/android_ipopt/releases, and extract them into the `project/app/libs` directory.

3. Open `project/app/build.gradle`. 

   In `android/defaultConfig`, add the ABIs that you want to compile for. For example, add the lines

   ```
   ndk {
       abiFilters 'x86', 'x86_64', 'arm64-v8a', 'armeabi'
   }
   ```

   In the video, I add the 4 ABIs above, but I really only need `x86_64` since that is the emulator I am testing in. However, in practice most new Android devices are `arm64-v8a`, so if you are going to put it on a real device, you probably want that option as well. Note that you can also add `mips` or `mips64`, but I have yet to see such a device in the wild.

   Next, in the main `android` section of the same file (outside of `defaultConfig`), add the lines

   ```
   sourceSets {
   	main {
   		// let gradle pack the shared library into apk
   		jniLibs.srcDirs = ['libs']
   	}
   }
   ```

   This is necessary so that the `.so` libraries get packaged with the app. 

4. Go into the `project/app/src/main/cpp` folder and copy in your C++ code. You have to be very careful about your method naming here. If you are going to call a method from the `stringFromJNI` method of `MainActivity` class located in the `io.jeti.ipopt` package, then the C++ method should have the signature

   ```
   extern "C" JNIEXPORT jstring
   JNICALL
   Java_io_jeti_ipopt_MainActivity_stringFromJNI(
       JNIEnv *env,
       jobject /* this */) {
       ...
       return ...
    }

   ```

   But I am not here to give you a primer on the JNI. If you don't know what I am talking about, or are not familiar with JNI naming schemes, then please find a JNI tutorial. 

5. Open `project/app/CMakeLists.txt`. 

   You need to update this file to reflect the C++ code that you actually want to build, and to make sure that you are linking to ipopt. 

   To accomplish the first part of this, find the `add_library` portion of the file. You want to change that portion so that it builds your code, not the default code that was created when you created the project. In my case, the auto-generated block looked like 

   ```
   add_library( # Sets the name of the library.
                native-lib

                # Sets the library as a shared library.
                SHARED

                # Provides a relative path to your source file(s).
                src/main/cpp/native-lib.cpp)
   ```

   However, we replaced the C++ code `src/main/cpp/native-lib.cpp` with our own files 

   - `src/main/cpp/cpp_example.cpp`
   - `src/main/cpp/MyNLP.cpp`

   Therefore we updated that block to read

   ```
   add_library( # Sets the name of the library.
                native-lib

                # Sets the library as a shared library.
                SHARED

                # Provides a relative path to your source file(s).
                src/main/cpp/cpp_example.cpp
                src/main/cpp/MyNLP.cpp)
   ```

   Finally, we had to link in the dependencies to IPOPT. Since IPOPT itself is dependent on some other libraries, we first need to tell cmake about our external libraries by adding lines of the form

   ```
   add_library(xyz SHARED IMPORTED)
   set_property(TARGET xyz PROPERTY IMPORTED_LOCATION  
                ${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libxyz.so)
   ```

   for each of our external libraries, that is, is for `xyz=blas,lapack,metis,mumps,ipopt`. Then, since the includes where all placed in `project/app/libs/include`, we tell cmake where to find them

   ```
   # Location of header files
   include_directories(${CMAKE_SOURCE_DIR}/libs/include/coin
                       ${CMAKE_SOURCE_DIR}/libs/include/coin/ThirdParty)
   ```

   Lastly, we link the libraries to our project by updating the `target_link_libraries` call from something like 

   ```
   target_link_libraries( # Specifies the target library.
                          native-lib

                          # Links the target library to the log library
                          # included in the NDK.
                          ${log-lib}
                          )
   ```

   to 

   ```
   target_link_libraries( # Specifies the target library.
                          native-lib

                          # Links the target library to the log library
                          # included in the NDK.
                          ${log-lib}

                          # Link to IPOPT
                          blas
                          lapack
                          metis
                          mumps
                          ipopt
                          )
   ```



Happy optimizing...
