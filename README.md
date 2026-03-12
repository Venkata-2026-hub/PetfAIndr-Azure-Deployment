# PetfAIndr-Azure-Deployment
Cloud Computing project deploying the PetfAIndr AI microservice application on Microsoft Azure using AKS, Docker, and Azure CLI.
**
**PetfAIndr – AI-Based Lost Pet Detection System on Microsoft Azure
Project Overview****

PetfAIndr is a cloud-based application designed to help communities locate lost pets using Artificial Intelligence and modern cloud technologies.
Pet owners can report missing pets, while community members can upload images of pets they find. The system analyzes these images using AI to determine if the pet matches any registered lost pet, increasing the chances of reconnecting pets with their owners.
This project demonstrates how a microservice architecture can be deployed on Microsoft Azure using containerization, Kubernetes, and AI services.
System Architecture
The PetfAIndr system follows a microservice architecture consisting of two containerized services.
Frontend
•	Built using Blazor Web
•	Provides the user interface
•	Allows users to report lost pets and upload found pet images
Backend
•	Built using Python
•	Handles business logic
•	Communicates with AI services
•	Stores and processes pet data
Both services run inside Docker containers and are deployed on Azure Kubernetes Service (AKS).
Cloud Services Used
The system uses several Azure services:
Service	Purpose
Azure Kubernetes Service (AKS)	Container orchestration
Azure Container Registry (ACR)	Store Docker images
Azure Cosmos DB	NoSQL database for pet information
Azure Storage Account	Stores uploaded pet images
Azure Service Bus	Messaging between services
Azure Custom Vision AI	AI model for image recognition

**Architecture Flow**
User → Frontend (Blazor)
Frontend → Backend (Python)
Backend communicates with:
•	Azure Cosmos DB
•	Azure Storage
•	Azure Service Bus
•	Azure Custom Vision AI
All services run inside Azure Kubernetes Service.
