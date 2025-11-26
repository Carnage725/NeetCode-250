# RECRUITER-APPLICANT SIMULATION SYSTEM - BUILD INSTRUCTIONS

## PART 1: PROJECT SETUP

### 1.1 Initialize Project Structure
Create a full-stack application with three main directories: backend (FastAPI), frontend (React + Vite), and uploads (for files).

### 1.2 Set Up Tech Stack
- Backend: FastAPI with Python 3.11+
- Frontend: React with Vite and Tailwind CSS
- Database: PostgreSQL 15
- Containerization: Docker and Docker Compose
- Media: Use browser MediaRecorder API for video/audio

---

## PART 2: DATABASE DESIGN

### 2.1 Create scenarios Table
Store assessment scenarios with fields: id (primary key), recruiter_id, title, description, created_at timestamp, is_active boolean.

### 2.2 Create tasks Table
Store individual tasks with fields: id (primary key), scenario_id (foreign key to scenarios), task_type (text/mcq/video/file/audio), question_text, order_number for sequencing, time_limit_seconds (nullable), is_skippable boolean, options_json for MCQ choices, scoring_rules_json for auto-scoring configuration, correct_answer for MCQs, max_score defaulting to 10.

### 2.3 Create applicants Table
Store applicant records with fields: id (primary key), scenario_id (foreign key), unique_link_token (unique string), name and email (nullable), started_at and completed_at timestamps (nullable), total_score decimal, status (invited/in_progress/completed).

### 2.4 Create responses Table
Store applicant answers with fields: id (primary key), applicant_id and task_id (foreign keys), answer_text, selected_option for MCQs, file_url, video_url, audio_url (all nullable), time_taken_seconds, score (nullable), submitted_at timestamp, is_complete boolean. Add unique constraint on (applicant_id, task_id).

---

## PART 3: BACKEND API - SETUP

### 3.1 Configure Database Connection
Set up SQLAlchemy with PostgreSQL connection using environment variable DATABASE_URL. Create session management with dependency injection pattern.

### 3.2 Define ORM Models
Create SQLAlchemy models matching all four database tables with proper relationships and cascade deletes.

### 3.3 Create Pydantic Schemas
Define request/response schemas for scenario creation, task creation, response submission, and all data transfer objects.

### 3.4 Install Dependencies
Include FastAPI, Uvicorn with standard extras, SQLAlchemy, psycopg2-binary, Pydantic, python-multipart for file uploads, python-jose for tokens.

---

## PART 4: BACKEND API - RECRUITER ENDPOINTS

### 4.1 POST /api/scenarios
Accept title, description, recruiter_id. Create new scenario in database. Return created scenario with id and timestamp.

### 4.2 GET /api/scenarios
Accept recruiter_id as query parameter. Fetch all scenarios for that recruiter. For each scenario, calculate applicant count and average score. Return enriched scenario list.

### 4.3 GET /api/scenarios/{scenario_id}
Fetch specific scenario with all its tasks ordered by order_number. Return scenario details plus tasks array.

### 4.4 POST /api/scenarios/{scenario_id}/tasks
Accept all task fields (type, question, options, scoring rules, etc.). Create task linked to scenario. Return created task.

### 4.5 PUT /api/tasks/{task_id}
Accept any task fields to update. Update existing task. Return updated task.

### 4.6 DELETE /api/tasks/{task_id}
Delete task by id. Tasks table has CASCADE delete on scenario deletion. Return success confirmation.

### 4.7 POST /api/scenarios/{scenario_id}/generate-link
Generate UUID v4 token. Create applicant record with token, scenario_id, and status 'invited'. Return token and full URL formatted as https://yourapp.com/apply/{token}.

### 4.8 GET /api/scenarios/{scenario_id}/applicants
Fetch all applicants for scenario. Return list with name, email, status, total_score, started_at, completed_at, calculated time_taken.

### 4.9 GET /api/applicants/{applicant_id}/responses
Fetch applicant details and all their responses joined with task information. Return applicant object and responses array with task questions and answers.

### 4.10 PUT /api/responses/{response_id}/score
Accept manual score value. Update response score. Recalculate applicant total_score. Return updated response.

---

## PART 5: BACKEND API - APPLICANT ENDPOINTS

### 5.1 GET /api/apply/{token}
Validate token exists in database. Fetch associated scenario and verify it's active. Count total tasks for scenario. Return scenario details (title, description), applicant_id, total_tasks count.

### 5.2 GET /api/apply/{token}/tasks
Validate token. Fetch all tasks for that scenario ordered by order_number. Return tasks array but exclude sensitive fields like correct_answer and detailed scoring_rules_json.

### 5.3 POST /api/apply/{token}/start
Validate token. Update applicant status to 'in_progress' and set started_at to current timestamp. Return applicant_id and new status.

### 5.4 POST /api/apply/{token}/responses
Validate token. Accept task_id, answer_text, selected_option, time_taken_seconds. Check if response already exists for this applicant-task pair. If exists, update it. If not, create new. Auto-calculate score for MCQs (compare selected_option to correct_answer) and text answers (keyword matching if scoring_rules_json contains keywords array). Return response_id and calculated score.

### 5.5 PUT /api/apply/{token}/responses/{response_id}
Accept updated answer fields and is_complete flag. Update existing response. Recalculate score if applicable. Return updated response.

### 5.6 POST /api/apply/{token}/complete
Validate token. Sum all response scores for this applicant. Update applicant with total_score, set completed_at timestamp, change status to 'completed'. Return total_score and completed_at.

---

## PART 6: BACKEND API - FILE UPLOAD

### 6.1 POST /api/upload
Accept multipart form data with fields: file (the uploaded file), applicant_id, task_id, file_type (video/audio/file). Create directory structure /uploads/{applicant_id}/ if not exists. Save file with naming pattern {task_id}_{timestamp}_{filename}. Store file in uploads directory. Return file_url as relative path like /uploads/{applicant_id}/{filename}.

### 6.2 Serve Static Files
Configure FastAPI to serve /uploads directory as static files so URLs are accessible.

---

## PART 7: FRONTEND - PROJECT SETUP

### 7.1 Initialize React Project
Create Vite project with React template. Install dependencies: react-router-dom for routing, axios for API calls, tailwindcss for styling, lucide-react for icons, @dnd-kit/core for drag-drop, @tanstack/react-table for tables.

### 7.2 Configure Routing
Set up React Router with two main route groups: recruiter portal (/login, /dashboard, /scenarios/*) and applicant portal (/apply/:token/*).

### 7.3 Create API Client
Set up axios instance with base URL pointing to backend. Configure default headers and response interceptors.

---

## PART 8: FRONTEND - RECRUITER PORTAL

### 8.1 Login Page (/login)
Create simple form with email and password inputs. On submit, authenticate user (implement basic auth or OAuth). Store auth token in localStorage. Redirect to dashboard on success.

### 8.2 Dashboard Page (/dashboard)
Fetch all scenarios for logged-in recruiter. Display as grid of cards showing title, created date, applicant count, average score. Include "Create New Scenario" button that navigates to scenario builder.

### 8.3 Scenario Builder Page (/scenarios/new)
Create form with title and description inputs. Display list of added tasks with drag-drop reordering capability using dnd-kit. Show "Add Task" button that opens modal. In modal, show dropdown to select task type (text/mcq/video/file/audio). Based on selection, show dynamic form. For text: question textarea, optional timer, skip toggle. For MCQ: question textarea, option inputs with add/remove buttons, correct answer dropdown, timer, skip toggle. For video/audio/file: question textarea, max file size input, timer, skip toggle. Allow editing scoring rules (manual vs auto, keywords for text matching). Save button should POST scenario then POST each task sequentially. Show "Generate Link" button that calls generate-link endpoint and displays copyable link in modal.

### 8.4 Applicant Dashboard Page (/scenarios/{id}/applicants)
Fetch all applicants for scenario. Display in table with columns: Name, Email, Status, Score, Time Taken, Started At, Completed At. Add status filter dropdown (all/invited/in_progress/completed). Make table rows clickable to navigate to response review page.

### 8.5 Response Review Page (/applicants/{id}/review)
Create two-column layout. Left sidebar shows list of all tasks (clickable). Main area shows current task question and applicant's response. For text responses, show in read-only textarea. For MCQ, show selected option with visual indicator if correct/incorrect. For video, embed video player with controls. For audio, embed audio player. For files, show download button. If response has no auto-score, show score input field and save button. Add Next/Previous buttons to navigate between tasks.

---

## PART 9: FRONTEND - APPLICANT PORTAL

### 9.1 Landing Page (/apply/{token})
On mount, call GET /api/apply/{token} to validate and fetch scenario. If invalid, show error page. If valid, display scenario title and description. Show instructions like "Your progress will be auto-saved. You can answer questions in any order if skipping is allowed." Add "Start Simulation" button that calls POST /api/apply/{token}/start then navigates to simulation flow.

### 9.2 Simulation Flow Page (/apply/{token}/simulation)
Fetch all tasks on mount. Store in state along with current task index. Display progress bar showing (current task number) / (total tasks) × 100%. If current task has time limit, show countdown timer that starts on task load. Display task question prominently. Render input based on task_type. For text: large textarea. For MCQ: radio button group with all options. For file: drag-drop zone with file input fallback. For video: button that opens video recording modal. For audio: button that opens audio recording modal. Implement auto-save with debouncing (save 2 seconds after last input change). On every auto-save, call POST /api/apply/{token}/responses with current answer. Show "Back" button (unless first task) that decrements task index. Show "Save & Continue" button that saves current response with is_complete=true, then increments task index. On last task, "Save & Continue" calls POST /api/apply/{token}/complete then navigates to completion page.

### 9.3 Video Recording Modal
Request camera and microphone permissions using navigator.mediaDevices.getUserMedia. Show live preview using video element with srcObject set to media stream. Add "Start Recording" button that initializes MediaRecorder and begins capture. Show "Stop Recording" button that stops MediaRecorder and stores recorded chunks. Show "Re-record" button that resets and allows new recording. Show "Confirm & Upload" button that creates Blob from chunks, creates FormData with blob, calls POST /api/upload with file_type='video', displays upload progress bar, and on success saves video_url to response.

### 9.4 Audio Recording Modal
Similar to video modal but request only audio using getUserMedia audio-only. Show recording indicator (red dot or waveform visualization). Same start/stop/re-record/upload flow as video.

### 9.5 Completion Page (/apply/{token}/complete)
Display large checkmark icon. Show "Submission Complete!" heading. Display thank you message. Show submission timestamp and total score if available. Do not provide option to go back.

---

## PART 10: CORE FEATURES IMPLEMENTATION

### 10.1 Auto-Save Mechanism
In simulation flow, use React useEffect hook that watches answer state. Set up debounced timeout of 2000ms. When answer changes, clear previous timeout and set new one. On timeout trigger, call save response API. Clean up timeout on component unmount.

### 10.2 Progress Persistence
When applicant refreshes page during simulation, fetch their existing responses from backend. Match responses to tasks by task_id. Pre-fill answers in form inputs. Resume from where they left off.

### 10.3 Auto-Scoring Logic
In backend when saving response, check task type. For MCQ: compare selected_option to task correct_answer, award full max_score if match, 0 if not. For text with keyword scoring: extract keywords array from scoring_rules_json, count how many keywords appear in answer_text (case-insensitive), calculate score as (matches / total_keywords) × max_score. For video/file/audio or text without rules: set score to null for manual review.

### 10.4 Link Generation
Use Python uuid.uuid4() to generate cryptographically secure random token. Store as string in applicants table unique_link_token field. Return formatted URL with token as path parameter.

### 10.5 Timer Functionality
In frontend, when task with time_limit_seconds loads, start countdown using setInterval. Display remaining time in MM:SS format. When timer reaches zero, auto-save current answer, mark as is_complete=true, and advance to next task automatically.

### 10.6 File Upload Handling
For file uploads, validate file size and type on frontend before upload. Use FormData to send multipart request. Show upload progress using axios onUploadProgress callback. On backend, validate file extensions, generate safe filenames, create directory structure, save file, return accessible URL path.

---

## PART 11: DOCKER DEPLOYMENT

### 11.1 Backend Dockerfile
Use tiangolo/uvicorn-gunicorn-fastapi:python3.11 as base image. Set working directory to /app. Copy requirements.txt and install dependencies with pip. Copy entire backend code. Expose port 8000. Set command to run uvicorn with host 0.0.0.0 and port 8000.

### 11.2 Frontend Dockerfile
Use multi-stage build. Stage 1: Use node:18-alpine, copy package files, run npm install, copy source code, run npm run build to create production bundle. Stage 2: Use nginx:alpine, copy built files from stage 1 to /usr/share/nginx/html, copy custom nginx configuration, expose port 80.

### 11.3 Nginx Configuration
Configure nginx to serve React app on port 80. Set up routing so all requests to /api/* proxy to backend service on port 8000. All other requests serve index.html for React Router to handle client-side routing.

### 11.4 Docker Compose File
Define three services: db (postgres:15 image with environment variables POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, volume mount for data persistence, healthcheck using pg_isready), backend (build from ./backend, expose 8000, environment variables for DATABASE_URL and UPLOAD_DIR, volume mount ./uploads to /app/uploads, depends on db service healthy), frontend (build from ./frontend, expose 3000 mapped to internal 80, depends on backend service).

### 11.5 Environment Variables
Backend needs: DATABASE_URL (postgres connection string), UPLOAD_DIR (path to uploads folder), SECRET_KEY (for JWT if implementing auth). Frontend needs: VITE_API_URL (backend API base URL).

---

## PART 12: BUILD SEQUENCE

### 12.1 Phase 1 - Database
Create PostgreSQL database. Run all table creation SQL scripts. Verify tables exist and relationships are correct. Test with sample insert queries.

### 12.2 Phase 2 - Backend Core
Set up FastAPI project structure. Configure database connection and ORM models. Create all API endpoints without complex logic first (just CRUD operations). Test each endpoint using Postman or Thunder Client. Verify data flows correctly.

### 12.3 Phase 3 - Backend Advanced
Implement auto-scoring logic in response save endpoint. Add file upload endpoint with proper file handling. Implement link generation with UUID. Test complete recruiter workflow: create scenario, add tasks, generate link.

### 12.4 Phase 4 - Frontend Recruiter Portal
Build login page and authentication flow. Create dashboard with scenario grid. Build scenario builder with task management. Implement drag-drop task reordering. Create applicant dashboard with table. Build response review page with media players. Test complete recruiter workflow in UI.

### 12.5 Phase 5 - Frontend Applicant Portal
Build landing page with token validation. Create simulation flow with progress tracking and timer. Implement all input types (text, MCQ, file upload). Build video and audio recording modals with MediaRecorder API. Implement auto-save with debouncing. Create completion page. Test complete applicant workflow.

### 12.6 Phase 6 - Integration Testing
Test end-to-end flow: recruiter creates scenario, generates link, applicant completes simulation, recruiter reviews responses. Fix any bugs or edge cases. Ensure auto-save works reliably. Verify file uploads and video recording work across browsers.

### 12.7 Phase 7 - Docker Deployment
Create Dockerfiles for backend and frontend. Write docker-compose.yml. Build images and test locally. Ensure volume mounts work for file persistence. Verify networking between containers. Test production build performance.

### 12.8 Phase 8 - Optional AI Evaluation
If adding AI scoring, create new endpoint POST /api/tasks/{task_id}/ai-evaluate. Send task question and applicant answer to OpenAI API. Request structured response with score 0-10 and feedback text. Store AI score and feedback in responses table. Add UI in response review to trigger AI evaluation and display results.

---

## PART 13: KEY FEATURES CHECKLIST

### 13.1 Must-Have Features
- Recruiter can create scenarios with multiple tasks
- Support 5 task types: text, MCQ, video, file, audio
- Task ordering and reordering capability
- Unique link generation for applicants
- Applicant simulation flow with progress tracking
- Auto-save every 2 seconds during simulation
- Task timer with auto-advance
- Video and audio recording in browser
- File upload with drag-drop
- Auto-scoring for MCQ and keyword-based text
- Manual scoring interface for recruiters
- Applicant dashboard with filtering
- Detailed response review with media playback

### 13.2 Nice-to-Have Features
- Branching logic (show different tasks based on previous answers)
- Skip functionality (allow applicants to skip tasks if enabled)
- AI-powered evaluation using OpenAI API
- Export results to CSV
- Email notifications when applicant completes
- Analytics dashboard with charts
- Multi-recruiter tenant support
- Applicant authentication (optional name/email collection)

### 13.3 UI/UX Requirements
- Clean, modern interface with Tailwind CSS
- Mobile-responsive design
- Loading states for all async operations
- Error handling with user-friendly messages
- Confirmation dialogs for destructive actions
- Toast notifications for success/error feedback
- Keyboard navigation support
- Accessible form labels and ARIA attributes

---

## PART 14: TESTING AND VALIDATION

### 14.1 Backend Testing
Test all API endpoints with valid and invalid inputs. Verify authentication and authorization. Check database constraints and foreign key relationships. Test file upload size limits and type validation. Verify auto-scoring accuracy.

### 14.2 Frontend Testing
Test all user flows end-to-end. Verify form validations. Test auto-save and progress persistence. Check video/audio recording across different browsers. Verify timer functionality. Test responsive design on mobile devices.

### 14.3 Integration Testing
Test complete recruiter-to-applicant flow. Verify data consistency across frontend and backend. Test concurrent applicant submissions. Check file storage and retrieval. Verify scoring calculations.

---

## FINAL NOTES

Build in order: Database → Backend APIs → Recruiter Portal → Applicant Portal → Docker → Testing → Deployment. Keep features simple and functional first, then add polish. Test frequently after each feature. Deploy early and iterate based on feedback. Prioritize core functionality over advanced features. Ensure auto-save works reliably as it's critical for user experience.
