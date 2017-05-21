# gRPC
```
protoc {
    path = '/usr/bin/protoc'
}
```
didn't work for me, I needed to:
```
../gradlew java_pluginExecutable -Pprotoc=/usr/bin/protoc
```



# Bazel
```
cd $BAZEL_HOME
sudo update-alternatives --config java
```
select: 
`/usr/lib/jvm/jdk-8-oracle-arm32-vfp-hflt/jre/bin/java`
```
git checkout 0.4.3
nano tmp.sh
```
in tmp.sh:
```
#!/bin/bash
export PROTOC=`which protoc`
export GRPC_JAVA_PLUGIN=../grpc-java/compiler/build/exe/java_plugin/protoc-gen-grpc-java
./compile.sh
```
```
chmod +x tmp.sh
sudo ./tmp.sh
```
then follow regular bazel steps

# Tensorflow
```
cd $TENSORFLOW_HOME
nano tensorflow/workspace.bzl
```
in workspace.bzl change:
`http://zlib.net/zlib-1.2.8.tar.gz`
to:
`http://zlib.net/fossils/zlib-1.2.8.tar.gz`

`bazel build -c opt --copt="-mfpu=neon-vfpv4" --copt="-funsafe-math-optimizations" --copt="-ftree-vectorize" --copt="-fomit-frame-pointer" --local_resources 1024,1.0,1.0 --verbose_failures tensorflow/tools/pip_package:build_pip_package`
