# Project Assignment 2: gRPC-Backed Image Search Engine
### Team Eswar 
## Creating necessary files
* To implement the multi-threaded server for the image search engine using gRPC and containerize it using Docker, we need to create necessary files first.
* Creating Protobuf File (.proto):

Creating a file named ```image_search.proto``` to define the service methods and message types:
```
syntax = "proto3";
option go_package = "./ImageSearch";
package image_search;
service ImageSearch {
    rpc SearchImage (SearchRequest) returns (ImageResponse) {}
}

message SearchRequest {
    string keyword = 1;
}

message ImageResponse {
    bytes image_data = 1;
}
```
 Compiling Protocol Buffers
```
 protoc --go_out=. --go-grpc_out=. image_search.proto
```
 
