# Face Recognition alarm System
The aim of the Face Recognition alarm System (FRAS) project is to implement an alarm detection system. The main objective is to identify if the person entering through the main door or garden is authorized or is an unknown intruder. In my case, this information is used by my home automation server, Home Assistant (HA).

# Hardware
The hardware used is the following:
- A server where all containers and moduls can run.
   In my case i use my Ugreen Nas with 8GB of RAM.
- Edge camera (TP-Link Tapo C220)
  In my case i use the TP-Link Tapo C220: the quality is 2K QHD (4MP) with an aperture of $f/2.0$; it has the native Real-Time Streaming Protocol (RTSP) dual stream, with the main stream in high-resolution and the sub-stream in low-resolution. In addition, it features on-device Smart AI Detection capable of distinguishing between general motion, persons, and pets directly on the edge.

# Software
The software architecture is service-oriented, based on Docker. All components are orchestrated with the docker-compose file over an internal virtual private network. The main components are:
- **[Frigate](https://github.com/blakeblackshear/frigate)** is an event-based Network Video Recorder with low latency. It is composed of different modules:
  - Multiplexer: *go2rtc* takes as input the RTSP stream of the camera and sends it to different clients. This is useful when we have more clients than camera channels.
  - It performs continuous frxèame-differencing motion detection.
  - It has an Object Detection model (i.e., *MobileDet* or *YOLO*) that classifies an entity into different base classes (e.g., Person, Dog, Car).
- **[Double Take](https://github.com/jakowenko/double-take)** is a middleware that orchestrates the entire pipeline in a computer vision environment.
  - It takes the correct input from the correct topic list in a broker. This is useful if we have more than one camera and we have more than one operation to do for each camera.
  - It performs multi-frame analysis over incoming events: it can sample multiple consecutive frames at short intervals (e.g., 200ms).
  - It asks for computation from one or more models and can perform a consensus operation (e.g., average) to increase the recall metrics. The challenge in this case will be to normalize multiple JSON results formatted in different ways.
  - It sends the result to one or more clients (e.g., HA).
- **[CompreFace](https://github.com/exadel-inc/CompreFace)** is a modular stack based on a microservices architecture:
   - Front-end (`compreface-front-end`): An Nginx-based web UI and reverse proxy / API gateway. It serves the dashboard to manage projects and face collections, and routes incoming traffic to the admin and API services.
   - Admin (`compreface-admin`): A Spring Boot service managing administrative operations, applications, users, and API key generation.
   - API (`compreface-api`): The backend service responsible for processing face recognition, detection, and verification requests, validating API keys, and communicating with the core engine.
   - Inference Engine (`compreface-core`): The deep learning engine that executes AI models at runtime. The recognition pipeline detects face bounding boxes, performs alignment, extracts 512-dimensional embeddings, and computes cosine similarity against stored vectors.
   - Database (`compreface-postgres-db`): A PostgreSQL database used to store users, applications, services, and facial embedding metadata.

# Data Flow
The camera data flow must be transformed into an output that can be useful for a client. This process defines various phases:
1. Input Multiplexing

   The *go2rtc* module of *Frigate* behaves as a proxy: it receive frame from the camera using both two channel, the main-stream (high-resolution channel with a high bitrate) and the sub-stram (low-resolution channel with a low bitrate); if more then one client want to read the camera frame (e.g., live camera control from smartphone), have to ask to this component.
2. Edge Pre-Filtering and Dynamic Wake-up
   Frigate's continuous detection is kept disabled by default: the TP-Link Tapo camera performs on-device AI classification: non-human movements (e.g., a pets) are discarded directly on the edge; when the camera detects a person, HA receives an immediate trigger and turns ON Frigate detection and recordings via MQTT. Once the person leaves the scene, Frigate detection is turned back OFF.
3. Multi-Frame Extraction and Orchestration

   Frigate tracks the person in the monitored zone and publishes events on the MQTT broker. Double Take intercepts the event and orchestrates a multi-frame retrieval: it fetches high resolution crops from Frigate at quick intervals (up to 15 attempts spaced by 0.2s) as the subject approaches, forwarding each crop to CompreFace.
4. Face Recognition

   CompreFace has the objective to recognize if the person who entered is authorized or not: *RetinaFace* isolates the face and detect 5 coordinates (eyes, nose, mouth corners); the backbone of the CNN *ArcFace* extracts the embeddings of 512 dimensions; computes the cosine similarity against all vectors in the database; send back to Double Take the maximum value found.
5. Verification and Alarm Decision

   Double Take compares the result with a given threshold:
   - If an authorized person (e.g., family members) is matched with confidence $\ge 0.8$, the check succeeds immediately and Double Take stops requesting further frames. Home Assistant marks the event as safe and suppresses the alarm.
   - If the multi-frame window expires without a recognized identity (or if the face remains unknown), Home Assistant flags the event as an unauthorized intrusion and triggers the alarm.

# Implementation
The system is deployed on a private server (i.e., my NAS running Debian x86_64) using Docker Compose to containerize the entire microservices pipeline. Deployment and lifecycle management are fully automated via a GitOps and CI/CD approach: every push to the `main` branch automatically triggers synchronization, container recreation, and environment updates on the NAS.

### Continuous Deployment
The continuous deployment is implemented throught a the GitHub Actions runner (`myoung34/github-runner`) that operates directly on my NAS inside Docker.

In this way:
  - There are no port-forwarding rules on my router: GitHub sends an HTTP POST to the GitHub Actions runner container and it becomes responsible for updating the containers that changed.
  - We garantee host OS isolation: Git tools and commands and other deployment tools live inside the dedicated container, leaving the host OS stable and immutable.

### Broker
A publish-subscribe broker is essential to avoid polling communication between Double Take and Frigate. My Server is a NAS running my HA server, where I already use an instance of Mosquitto broker. To avoid recreating a new container running Mosquitto, I used the HA instance during the pipeline of FRAS.