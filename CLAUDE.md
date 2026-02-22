# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Run Commands

```bash
# Build the project
./mvnw clean install

# Run the application
./mvnw spring-boot:run

# Run all tests
./mvnw test

# Run a specific test class
./mvnw test -Dtest=SpringBatchLabApplicationTests

# Run a specific test method
./mvnw test -Dtest=SpringBatchLabApplicationTests#contextLoads

# Package as JAR
./mvnw clean package

# Build Docker image (Spring Boot Buildpacks)
./mvnw spring-boot:build-image
```

## Project Overview

Spring Batch Lab is a Spring Boot 3.5.11 / Java 17 application for building batch processing workflows. It uses:

- **Spring Batch** — core batch processing framework (jobs, steps, readers, processors, writers)
- **Spring Boot Actuator** — health and metrics endpoints
- **H2** — in-memory database used by Spring Batch for job/step metadata persistence
- **spring-batch-test** — utilities for testing batch jobs (e.g., `JobLauncherTestUtils`)

## Architecture

Spring Batch jobs follow the pattern:

```
Job → Step(s) → ItemReader → ItemProcessor → ItemWriter
```

The H2 schema is auto-created by Spring Batch on startup to store job execution metadata (`BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION`, etc.).

The entry point is `SpringBatchLabApplication.java`. Configuration for jobs and steps should be added as `@Configuration` classes under `com.mendonca.spring_batch_lab`.

There is no linting or code formatting tool configured in this project.
