# Assignment — Deploy a Three-Tier Application with Docker Compose

**Goal:** Containerize a simple three-tier app and run it end-to-end with Docker Compose. You’ll create Dockerfiles for the frontend and backend, define a Compose file for all services, and verify everything works together.

### Application Stack

Code Repository link - [Docker Assignment](https://github.com/saurabhd2106/docker-assignment-ih.git)

* Frontend: Node.js app (HTML/CSS/JS)
* Backend: Node.js API
* Database: PostgreSQL

### What You’ll Do?

1. Get the code: Clone the frontend and backend repositories.
2. Containerize the apps:
3. Create a Dockerfile for the frontend.
4. Create a Dockerfile for the backend.
5. Build images: Build the frontend and backend Docker images locally.
6. (Optional) Publish: Push both images to your Docker Hub registry.
7. Compose the stack: Create a docker-compose.yml that defines:
8. A frontend service (uses your frontend image)
9. A backend service (uses your backend image)
10. A database service (PostgreSQL)
11. A named volume for persistent database storage
12. A user-defined network so services can talk to each other
13. Necessary environment variables (e.g., DB name/user/password) and ports
14. Run the app: Use Docker Compose to start all services.
15. Verify end-to-end: Ensure the frontend loads, it can call the backend API, and the backend can read/write data in PostgreSQL.

### Success Criteria

* All containers are running and healthy.
* Frontend is accessible in the browser.
* Backend API responds successfully.
* Data persists after restarting containers (thanks to the database volume).

### Deliverables

* Dockerfile for the frontend
* Dockerfile for the backend
* docker-compose.yml
* Brief notes or screenshots showing the running app and verification steps\
  <br>

<br>
