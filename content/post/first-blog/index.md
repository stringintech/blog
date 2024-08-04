---
title: Overview and Design of a Distributed Task Manager
date: 2024-08-04 00:00:00+0000
image: cover.png
tags:
  - Software Design
weight: 1
---

## Introduction

The idea for this project originated from my desire to apply and challenge my knowledge as a software engineer with three years of experience on a personal project. This is an opportunity to share insights with others and learn more gradually. The project allows me to deepen my existing knowledge of software design and programming while also learning new things. It’s a perfect opportunity to rethink and challenge what I already know.

I am open to making changes that may significantly alter the structure or even the main purpose of the project. Since the primary goal is the learning process, I may adjust the project goals to explore different areas and learn more. As a result, this project might spawn sub-projects, hopefully creating components that are easily pluggable into other applications.

In this blog post, I will lay out the overall design for the project, which includes two main components: one for handling the creation and management of tasks called **Task Service** and another for handling notifications around task operations called **Notification Service**. The first component is developed in Go, a language I am comfortable with, while the second is developed in Rust, which is new to me and introduces new challenges.

## Design Goals

The primary goal of the design is to achieve a neat separation of concerns. This approach ensures that each component is modular, flexible, and scalable. By decoupling the core logic from specific implementations, I can easily switch or extend functionalities without impacting the entire system.

## Task Service

The Task Service is responsible for managing tasks. Here’s an overview of its architecture:

- **Core Component**: `TaskService` manages the main business logic related to tasks, regardless of the API system (REST, gRPC, or other) that may utilize it.
- **`StorageService` Interface**: Abstracts the storage mechanism for tasks, with possible implementations like PostgreSQL, AWS S3, etc. This allows the `TaskService` to interact with various storage backends without needing to know the specifics of each.
- **`EventEmitterService` Interface**: Manages the publishing of task-related events. As an example, one implementation might prepare events and send them to a RabbitMQ server. This decouples the `TaskService` from the specifics of event delivery, allowing different implementations as needed.

## Notification Service

The Notification Service listens for events from the Task Service and processes them in various ways. Here’s an overview of its architecture:

- **Core Component**: `EventHandlerService` manages the main event handling logic, regardless of the sources of events, which could be from a RabbitMQ server or other event sources.
- **`NotificationService` Trait**: Handles the delivery of notifications, with various possible implementations (e.g., email, Telegram). This allows different notification methods to be added or changed without affecting the core event handling logic.
- **`StorageService` Trait**: Abstracts the storage mechanism for events or notifications, with implementations like PostgreSQL or other storage solutions. This ensures that the Notification Service can store event data in various backends as needed.


## Repository Structure

Initially, both services are contained in a single [repository](https://github.com/stringintech/task-broker) to facilitate frequent related changes. As the project grows, I plan to separate them into individual repositories, allowing for more focused development and maintenance. This structure supports easier version control and dependency management during the early stages.

## Next Steps

In the next blog post, I will be:
- Defining the interfaces for the **Task Service**.
- Defining the traits for the **Notification Service**.
- Providing at least one implementation for each interface and trait.
- Adding an integration test (using test-containers) that shows how the two services work together.

## Conclusion

This project aims to build a robust and flexible distributed task manager. The design focuses on separation of concerns, modularity, and scalability. As the project progresses, I will continue to refine the architecture and expand the capabilities of both the Task Service and the Notification Service.