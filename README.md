
### Project Instructions: Secure Authentication with an Isolated Hashing Service

**Objective:**
Your goal is to build a secure user registration system with a strict security boundary. You will learn to orchestrate services that communicate over both a **`bridge` network** and a **shared Docker Volume**, demonstrating how to make a network-isolated service functional.

**Project Scenario:**
You will build a user registration system where the component that handles password hashing is completely isolated from the network. The `Frontend` will talk to an `Auth Service` over the network as usual. However, the `Auth Service` will securely pass work to the `Isolated Hasher` by dropping files into a shared directory, which acts as a job queue.

**Data Flow:**

1.  A user submits a `username` and `password` via the **Frontend Service**.
2.  The Frontend sends the credentials over the network to the **Auth Service**.
3.  The Auth Service checks a final `users.json` file to see if the user already exists. If so, it returns an error.
4.  If the user is new, the Auth Service **does not** hash the password. Instead, it writes a new "job" file (e.g., `request_123.json`) containing the username and plaintext password into a special `requests` directory within a shared volume.
5.  The **Isolated Hasher Service**, which is constantly watching this `requests` directory, detects the new file.
6.  The Hasher reads the file, hashes the password, then appends the new user record (with the **hashed password**) to the final `users.json` file located in the same shared volume.
7.  After successfully saving the new record, the Hasher deletes the original job file from the `requests` directory to mark the task as complete.

#### Application Architecture

1.  **The Frontend Service (Python Web App):**
    * Serves the user-facing HTML form and sends the registration request to the Auth Service over the network. Its role is unchanged.

2.  **The Auth Service (Python API):**
    * The "gatekeeper" of the system. It's the only service the frontend interacts with.
    * It exposes a `/register` endpoint.
    * Its primary jobs are to validate requests (e.g., check for existing users) and to create job files for the Hasher. It **does not** perform any hashing.

3.  **The Isolated Hasher Service (File-Based Worker):**
    * **This service must be on a `none` network.**
    * It has no API and cannot make or receive any network calls.
    * Its only connection to the outside world is the shared Docker Volume.
    * It runs a continuous loop to watch for new files in the `requests` directory. You can implement this with a simple `os.listdir` loop or a more advanced library like `watchdog`.

#### Key Docker Concepts to Implement

* **Mixed Networking:** You will use both a `bridge` network and a `none` network.
* **Volume as a Job Queue:** You will use a single named Docker Volume as a sophisticated data exchange medium, not just for persistence. It will contain the final user data *and* the temporary job queue directories.

#### Task Breakdown

1.  **Develop the Python Services:**
    * **Frontend Service:** Create a FastAPI app that serves an HTML registration form and makes a `POST` request to the `auth-service`.
    * **Auth Service:** Create a FastAPI app with a `/register` endpoint. This service needs logic to:
        * Check for existing users in a final data file (e.g., `/data/users.json`).
        * If the user is new, write a JSON file containing the username and password to a subdirectory within its mounted volume (e.g., `/data/requests/some_unique_id.json`).
    * **Isolated Hasher Service:** Write a Python script (no web framework needed, you can also implement any logic here) that:
        * Scans a directory (e.g., `/data/requests/`) for files.
        * For each file it finds, it reads the content, hashes the password (using a library like `bcrypt`), appends the new record to the final data file (`/data/users.json`), and then deletes the job file it just processed.

2.  **Containerize Your Services:**
    * Write a `Dockerfile` for each of the three applications, installing all necessary dependencies.

3.  **Orchestrate with Docker Compose:**
    * Create your `docker-compose.yml` file.
    * **Define Networks:** Create two networks: `app-net` (`bridge` driver) and `secure-net` (`none` driver).
    * **Define Volume:** Create one named volume: `job-data`.
    * **Define Services:**
        * Configure the `frontend-service` and `auth-service`. Connect both **only** to the `app-net` network.
        * Configure the `isolated-hasher-service`. Connect it **only** to the `secure-net`.
        * **Mount the Volume:** Mount the `job-data` volume to **both** the `auth-service` and the `isolated-hasher-service` at the same path (e.g., `/data`).

#### Verification Criteria

1.  The registration form works, and submitting a new user returns a "request received" message.
2.  The logs for the `auth-service` show it creating job files.
3.  The logs for the `isolated-hasher-service` show it detecting files, hashing passwords, and completing jobs.
4.  The final `users.json` file is correctly updated with the new user and their **hashed password**.
5.  The `isolated-hasher-service` is verifiably isolated and fails if you try to add any networking code to it.
6.  All user data persists across application restarts (`docker-compose down` and `docker-compose up`).

#### Submission Requirements

* A complete project directory with all source code, `Dockerfile`s, and the `docker-compose.yml`.
* A `README.md` file explaining your architecture. You must specifically describe how the `auth-service` and `isolated-hasher-service` communicate without a direct network link.
