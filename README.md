# ShopEase

ShopEase is a deployment-focused e-commerce application developed as the
workload for a cloud and infrastructure deployment project.

The application provides realistic customer and administrator workflows,
including product browsing, authentication, wishlist, cart, checkout,
order placement, order lifecycle management, and administration.

## Purpose

The application was built to provide a realistic workload for experimenting
with and documenting different deployment and infrastructure approaches.

The project will be deployed and tested using multiple approaches over time,
including AWS infrastructure, Docker, and Kubernetes.

## Technology Stack

- Python
- Flask
- HTML / CSS / JavaScript
- MySQL
- SQLAlchemy
- PyMySQL
- Amazon RDS
- Amazon S3
- Git / GitHub
- k6 for traffic and load testing

## Application Documentation

For a detailed explanation of the application:

[Application Documentation](docs/application/PROJECT_DOCUMENTATION.md)

## Deployment Documentation

For a detailed explanation of the deployment:

[Deployment Documentation](docs/deployment/DEPLOYMENT.md)

### Docker

Docker deployment documentation will be added later.

### Kubernetes

Kubernetes deployment documentation will be added later.

## Repository Structure

```text
app/        → Application source code
database/   → Database structure
static/     → CSS, JavaScript and static assets
templates/  → Application HTML templates
docs/       → Project and deployment documentation
