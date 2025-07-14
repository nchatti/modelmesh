# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Test Commands

- **Build with dependencies**: `mvn package -DskipTests` or `mvn install -DskipTests`
- **Run all tests**: `mvn test` (requires etcd to be running)
- **Run specific test**: `mvn test -Dtest=ModelMeshErrorPropagationTest`
- **Run multiple tests**: `mvn test -Dtest=SidecarModelMeshTest,ModelMeshFailureExpiryTest`
- **Quiet test output**: `mvn test -q`

## Prerequisites

- Java 21 (configured in Maven compiler plugin)
- Maven 3.x
- etcd (required for running tests)

## Architecture Overview

ModelMesh is a Java-based model serving management/routing layer that acts as a distributed LRU cache for ML models. Key architectural components:

### Core Components
- **ModelMesh**: Main orchestrator class (`src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`)
- **SidecarModelMesh**: Sidecar implementation for Kubernetes deployments
- **VModelManager**: Virtual model management system
- **ModelLoader**: Handles model loading and caching
- **TypeConstraintManager**: Manages deployment constraints

### Communication Layer
- **gRPC**: Primary communication protocol (protobuf definitions in `src/main/proto/`)
- **Thrift**: Legacy communication support (`src/main/thrift/`)
- **REST Proxy**: HTTP API support via separate proxy component

### Storage and Coordination
- **etcd**: Used for cluster coordination and configuration storage
- **Zookeeper**: Alternative coordination backend
- **Key-Value utilities**: Abstraction layer for distributed storage (`kv-utils`)

### Metrics and Monitoring
- **Prometheus**: Custom metrics collection with optimized client extensions
- **StatsD**: Alternative metrics backend
- **Netty**: Custom memory pool exports for monitoring

### Payload Processing
- **PayloadProcessor**: Extensible system for request/response processing
- **AsyncPayloadProcessor**: Asynchronous processing support
- **LoggingPayloadProcessor**: Request logging capabilities

## Source Generation

The project uses protobuf/gRPC code generation. Generated sources are placed in `target/generated-sources/protobuf/`. Run `mvn package -DskipTests` to generate sources before IDE setup.

## Test Structure

Tests are organized by functionality:
- **Core tests**: Basic ModelMesh functionality
- **Cluster tests**: Multi-instance deployment scenarios  
- **TLS tests**: Security and encryption
- **Payload tests**: Request processing validation
- **Integration tests**: End-to-end scenarios with different runtimes

## Development Notes

- Java 21 is the target version
- Uses custom optimized Prometheus client extensions
- Extensive use of Guava utilities for collections and concurrency
- Netty for high-performance networking
- Log4j2 for structured logging with JSON layout support