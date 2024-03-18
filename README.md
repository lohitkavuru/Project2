# Project Assignment 2: gRPC-Backed Image Search Engine
### Team Lohit
## Create necessary files
* To implement the multi-threaded server for the image search engine using gRPC and containerize it using Docker, we need to create necessary files first.
* Create Protobuf File (.proto):

Create a file named ```image_search.proto``` to define the service methods and message types:
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
This will generate ```image_search.pb.go``` and ```image_search_grpc.pb.go``` for Go, and you'll need to install the ```grpcio-tools``` package for Python.
 
## Implement Server (Python): (server.py)
```
import grpc
from concurrent import futures
import image_search_pb2
import image_search_pb2_grpc
import os
import random

class ImageSearchServicer(image_search_pb2_grpc.ImageSearchServicer):
    def SearchImage(self, request, context):
        image_dir = '/app/dataset/' + request.keyword
        image_files = os.listdir(image_dir)
        print(image_files)
        image_file = random.choice(image_files)
        with open(os.path.join(image_dir, image_file), 'rb') as f:
            image_data = f.read()
        return image_search_pb2.ImageResponse(image_data=image_data)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    image_search_pb2_grpc.add_ImageSearchServicer_to_server(ImageSearchServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```
## Implement Client (Python): (client.py)
```
import grpc
import image_search_pb2
import image_search_pb2_grpc
import os

def run():
    keyword = input("Enter keyword: ")
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = image_search_pb2_grpc.ImageSearchStub(channel)
        response = stub.SearchImage(image_search_pb2.SearchRequest(keyword=keyword))
        if response.image_data:
            filename = f'{keyword}'+'.jpg'
            with open(filename, 'wb') as f:
                f.write(response.image_data)
            print(f"Image received successfully for keyword '{keyword}'.")
            open_image(filename)
        else:
            print(f"No image found for the keyword '{keyword}'.")

def open_image(filename):
    try:
        # Open the image file using the default image viewer
        os.startfile(filename)
    except Exception as e:
        print(f"Error opening image file: {e}")

if __name__ == '__main__':
    run()

```
## Implement Client (GO) : (client.go)
```
package main

import (
	"bufio"
	"context"
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"os/exec"
	"strings"

	pb "image_search_engine" // Import the generated protobuf package

	"google.golang.org/grpc"
)

func run(keyword string, client pb.ImageSearchClient) {
	// Create a context and make the RPC call to the server
	ctx := context.Background()
	response, err := client.SearchImage(ctx, &pb.SearchRequest{Keyword: keyword})
	if err != nil {
		log.Fatalf("Error calling SearchImage: %v", err)
	}

	// Process the response
	if len(response.GetImageData()) > 0 {
		// Save the image data to a file
		filename := keyword + ".jpg"
		if err := ioutil.WriteFile(filename, response.GetImageData(), 0644); err != nil {
			log.Fatalf("Error writing image data to file: %v", err)
		}
		log.Printf("Image received successfully for keyword '%s'", keyword)

		// Open the image file
		if err := openFile(filename); err != nil {
			log.Printf("Error opening image file: %v", err)
		}
	} else {
		log.Printf("No image found for the keyword '%s'", keyword)
	}
}

func openFile(filename string) error {
	cmd := exec.Command("explorer", filename)

	if err := cmd.Run(); err != nil {
		return err
	}
	return nil
}

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Failed to dial server: %v", err)
	}
	defer conn.Close()

	// Create a client stub
	client := pb.NewImageSearchClient(conn)

	// Prompt the user to enter the keyword
	fmt.Print("Enter keyword: ")
	scanner := bufio.NewScanner(os.Stdin)
	scanner.Scan()
	keyword := strings.TrimSpace(scanner.Text())

	if keyword == "" {
		log.Fatal("Keyword cannot be empty")
	}

	run(keyword, client)
}

```

## Dockerize the Server
Create a Dockerfile (Dockerfile) to containerize the server:
```
# Use the official Python image as the base image for server and Python client
FROM python:3.8-slim AS server_python_client

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file into the container
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the server Python script and generated proto files into the container
COPY server.py .
COPY image_search_pb2.py .
COPY image_search_pb2_grpc.py .

# Expose the port on which the gRPC server will listen
EXPOSE 50051

# Command to run the server when the container starts
CMD ["python", "server.py"]

# Use the official Golang image as the base image for Go client
FROM golang:1.17 AS client_go

# Set the working directory in the container
WORKDIR /go/src/app

# Copy the Go client source code into the container
COPY client.go .

# Install the grpc package
RUN go get -u google.golang.org/grpc

# Build the Go client binary
RUN go build -o client client.go
# Command to run the Go client
CMD ["go", "run", "client.go"]

# Use Python image as the base image for Python client
FROM python:3.8-slim AS client_python

# Set the working directory in the container
WORKDIR /app

# Copy the Python client script and generated proto files into the container
COPY client.py .
COPY image_search_pb2.py .
COPY image_search_pb2_grpc.py .

# Install grpcio and protobuf packages
RUN pip install --no-cache-dir grpcio protobuf

# Command to run the Python client
CMD ["python", "client.py"]
```
Create a ````requirements.txt```` file if you have any dependencies.

## Build the Docker image:
```
docker build -t image_search_server .
```
## Run the Docker container with the dataset volume mounted:
```
docker run -d --name image_search_server -v \Users\eswarganipisetty\dataset:/app/dataset -p 50051:50051 image_search_server

```
## Run Python Client:
```
python client.py
```
## Run Go Client:
```
go run client.go
```
