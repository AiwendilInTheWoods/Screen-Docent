### Product Requirements Document (PRD) - Phase 1: Core Display Engine

#### 1. Objective
Build a robust, hardware-agnostic artwork display application that serves high-resolution image playlists to any modern web browser. Phase 1 focuses strictly on foundational media management, fluid front-end rendering, and advanced aspect ratio handling.

#### 2. Architecture Stack
* **Backend:** Python 3.11+ using **FastAPI**. It will serve local high-res image files, manage playlist configurations (stored as JSON), and provide REST endpoints for the frontend.
* **Frontend:** Lightweight HTML/CSS/JavaScript (or a minimal React/Vue setup if preferred). It will run in the browser, ping the backend for the next image in the playlist, and handle all visual transitions locally using the browser's GPU acceleration.

#### 3. Core Features (MVP)
* **Local Directory Ingestion:** The backend must scan a designated local folder structure, treating subfolders as distinct "Playlists" (e.g., `Abstract`, `Vintage Advertising`, `Landscapes`).
* **Playlist Management:** The backend will maintain the state of the current playlist, shuffling logic, and the timing intervals between images.
* **The Display Client (Frontend):** A distraction-free, full-screen web interface that smoothly crossfades between images received from the backend.

#### 4. Aspect Ratio & Rendering Engine
The frontend must provide the user with specific rendering configurations to handle the mismatch between the image's original aspect ratio and the display's aspect ratio. 
* **Mode A: The Ken Burns Pan.** The image scales to fill the screen entirely (no black bars). The frontend uses CSS/Canvas animations to slowly pan across the image over the duration of its display time, ensuring the entirety of the artwork is seen.
* **Mode B: Static User-Defined Crop.** The user can pre-define (via a configuration file or a basic admin UI) the specific X/Y coordinates of a bounding box for an image. The app will lock onto that specific framed area when displaying.
* **Mode C: Contain (Fallback).** The image is scaled to fit within the screen without cropping, displaying blurred, color-matched padding in the empty space (matte effect).

#### 5. Phase 1 Exclusions (Out of Scope)
* No LLM or agentic tagging integration.
* No external API calls for artwork metadata.
* No dynamic audio generation.

### Phase 2: The Administration Console
**Objective:** Build a dedicated web-based management dashboard to handle media ingestion, playlist organization, and metadata configuration.

**Data Layer:**
- Implement a local SQLite database using SQLAlchemy.
- Create models for `Playlist` (id, name) and `Artwork` (id, filename, playlist_id, crop_x, crop_y, crop_width, crop_height).

**Backend (FastAPI) Requirements:**
- Serve a new static file `admin.html` at the `/admin` route.
- Implement CRUD endpoints for uploading images (saving to the file system and DB), deleting images, creating/deleting playlists, and updating the crop metadata for specific artwork.

**Frontend (Admin Client) Requirements:**
- A clean, grid-based dashboard UI.
- A drag-and-drop zone for uploading new artwork to specific playlists.
- A visual cropping modal utilizing a Javascript cropping library (e.g., Cropper.js) to allow the user to define the exact static crop box for an image. This data must be saved back to the SQLite database via the API.
