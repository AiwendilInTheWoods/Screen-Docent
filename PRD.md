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

### Phase 3: The Curation Engine & AI Pipeline
**Objective:** Implement an autonomous, multi-agent pipeline to analyze artwork, generate metadata, and present it via a "Museum Placard" UI, gated by a Human-in-the-Loop (HITL) review system.

**1. Data Layer Expansion:**
- Update the `Artwork` table in `models.py` to include: `title`, `artist`, `year`, `description`, `tags` (JSON string), and `status` (default to 'pending_review', then 'approved').

**2. The Agentic Pipeline (Backend):**
- Create a new module `agents.py`.
- **Vision Agent:** Uses a local vision model (e.g., Ollama/LLaVA) or a lightweight API to generate visual and structural tags from the image array.
- **OSINT Researcher Agent:** Takes the image/visual data and queries open sources to identify the piece, artist, and year.
- **Scribe Agent:** Synthesizes the data into a concise, 2-sentence historical or artistic description.
- **Orchestrator:** A FastAPI background task triggered on upload that routes the image through these three agents and saves the output to the database.

**3. Human-in-the-Loop (Admin UI):**
- Update `admin.html` with a "Review Queue" tab.
- Display images with `status='pending_review'` alongside their AI-generated metadata.
- Provide inputs for the user to edit the data and an "Approve" button to change the status to 'approved', pushing it to the live playlists.

**4. The Museum Placard (Display Client):**
- Update `index.html` and `app.js` to read the new metadata.
- When the control overlay is active, display a dark glass-morphism panel on the left containing the Title, Artist, Description, and Tags.
- Generate a dynamic QR code in the UI (using a JS library like `qrcode.js`) that links to a Google search for the Artist + Title.

### Phase 4: Control, Scale, & Polish
**Objective:** Transition the application into a production-ready, multi-display platform with advanced remote management, persistent state, and refined AI curation.

**1. Remote Management & WebSocket Routing:**
- **WebSocket Remote Control:** Implementation of a mobile-first PWA remote (`/remote`) that provides instant, no-refresh DOM manipulation across connected displays.
- **Targeted Display Routing:** Introduction of display IDs via URL parameters (`?display=ID`), allowing a single server instance to manage multiple independent TVs with unique playlists and settings.

**2. Configuration & State Persistence:**
- **Hierarchical Configuration:** A robust override system where URL parameters supersede Playlist-level defaults, which in turn supersede Global system defaults. All time-based settings (cycle times, placard timers) are handled in seconds for user clarity.
- **State Persistence & Migrations:** Implementation of safe SQLite schema updates. On startup, the system automatically detects and adds missing columns to existing databases.
- **Docker Persistence:** Official `docker-compose` support with robust directory mapping for both the `/app/data` (database) and the `/app/Artwork` (media) directories to prevent data loss.

**3. Enhanced AI Curation:**
- **Context-Aware Analysis:** The AI Curator Agent now uses original filenames (cleaned of underscores/dashes) as contextual hints for identification.
- **Human-in-the-Loop (HITL) Regeneration:** Support for user-provided "AI Guidance" hints in the Review Queue, allowing users to correct AI hallucinations and regenerate placard data on-the-fly.

**4. Visual Refinement & Branding:**
- **Official Branding:** Rebranding of the platform as **Screen Docent**, including an integrated SVG logo and a cohesive "glass-morphism" aesthetic across all interfaces.
- **Responsive TV UI:** Optimized rendering for high-resolution displays, using `vmin` typography and strict viewport constraints to ensure a premium "Museum" look on various hardware (Fire TV, Android TV, etc.).
- Unobtrusive Logo Integration: Subtle logo placement on placards and hover controls to maintain the focus on the artwork.

### Phase 5: Semantic Discovery & RAG
**Objective:** Enhance the curation engine with factual external data and implement an automated discovery system for expanding the local library with world-class public domain art.

**1. Autonomous RAG Curator:**
- **Retroactive Enrichment:** A new pipeline that cross-references existing artwork titles and artists with the Wikipedia API.
- **Fact-Checked Metadata:** Uses verified Wikipedia summaries as "ground truth" context for the Gemini model to rewrite placard descriptions, eliminating AI hallucinations and ensuring historical accuracy.
- **Batch Processing:** Support for library-wide enrichment with integrated rate-limiting.

**2. Modular Semantic Art Scout:**
- **Scout Aggregator:** A pluggable architecture for hitting various museum and archive APIs.
- **ChicagoArtScout:** The first implementation, integrated with the Art Institute of Chicago's API.
- **High-Res Filters:** Automated filtering for high-resolution images (min 2000px) and explicit public-domain (CC0) licensing.
- **Discovery Queue:** A new "inbox" for approved scouted art, allowing users to preview thumbnails before downloading full-res files to their local library.

