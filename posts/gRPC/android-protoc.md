---
date: 2015-03-13
layout:  post
title:  Android gRPC protobuf的compile&generate问题
tagline:  ""
categories:
-  gRPC
tags:
- Android
- protobuf
---

Android上使用gRPC时，proto的compile和generate不能用java的方法，要用javanano的，而在文档中并没有给出具体的  command line

我找了很久终于发现了一种方法

先用

    protoc --proto_path="IMPORT_PATH" --javanano_out=ignore_services=true:"DST_DIR" "path/to/file.proto"    
    
生成 file.java 这是proto的类文件


再用(这里假设你在grpc-java的根目录下）

    protoc --plugin=protoc-gen-java_rpc=compiler/build/binaries/java_pluginExecutable/java_plugin --java_rpc_out=nano=true:"$OUTPUT_FILE" --proto_path="$DIR_OF_PROTO_FILE" "$PROTO_FILE"

生成xxxGrpc.java 这是用于Client的interfaces

也可以将这两个合起来使用(""中为自己规定）

    protoc --plugin=protoc-gen-java_rpc=compiler/build/binaries/java_pluginExecutable/java_plugin --java_rpc_out=nano=true:"examples/build/generated-sources/main/" --proto_path="examples/src/main/proto/" --javanano_out=ignore_services=true:"examples/build/generated-sources/main/" "examples/src/main/proto/helloworld.proto"
    
参考

[https://github.com/grpc/grpc-java/tree/master/compiler](https://github.com/grpc/grpc-java/tree/master/compiler)
 
[https://developers.google.com/protocol-buffers/docs/proto#generating](https://developers.google.com/protocol-buffers/docs/proto#generating)   
     
