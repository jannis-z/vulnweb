# Vulnweb

Vulnweb is a vulnerable web application intended as a capture the flag security challenge. The goal is to get access to the admin user and obtain the flag.

The application consists of a frontend, backend and database. These three services are installed by spinning up three Docker containers with docker-compose.

## Installation

Make sure you have Docker installed and running.

1. Clone this repository
```bash
git clone {url}
cd vulnweb
```
2. Start the containers with docker-compose
```bash
docker-compose up
```
3. Open the application at http://localhost:8080/

## Rules

While attempting the challenge you are not allowed to
- directly interact with the Docker containers (Docker CLI)
- use any credentials found in the docker-compose.yml file

## Walkthrough

If you are a beginner or do not have sufficient knowledge of web related vulnerabilities, you can use the [Walkthrough](Walkthrough.md) to guide you through the challenge.

## Vulnerability Summary

Check out the [Vulnerability Summary](VulnerabilitySummary.md) if you would like to see how these vulnerabilities were implemented and how they could be fixed.