---
title: GRPC mac istall protoc and generate Java code
date: 2024-06-26T14:24:23+08:00
draft: false
---

## 1. install protobuf

```
brew install protobuf
```

## 2. generate code with proto file
```
protoc --java_out=./ helloworld.proto
```

## 3. install protoc-gen-grpc-java
```
install protoc-gen-grpc-java from meven: 
    https://repo.maven.apache.org/maven2/io/grpc/protoc-gen-grpc-java/
download your version:
    like:  protoc-gen-grpc-java-1.64.0-osx-aarch_64.exe
```

## 4. chmod
need to set executable permissions
```
chmod +x protoc-gen-grpc-java-1.64.0-osx-aarch_64.exe
```

## 5. generate grpc service class
```
 protoc --plugin=protoc-gen-grpc-java=/your-path/protoc-gen-grpc-java-1.64.0-osx-aarch_64.exe --grpc-java_out=./ helloworld.proto
```