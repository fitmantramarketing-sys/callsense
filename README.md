## Callsense Project Analysis Report
Callsense is a React/Vite application that provides AI-powered analysis of sales calls. It allows users (sales agents and administrators) to upload audio recordings or PDF documents and receive detailed call transcripts, performance metrics, and coaching feedback. A live copilot(Partially implemented, not ready to use yet.) feature uses Google’s Gemini AI (via the @google/genai SDK) to provide real-time suggestions during calls. Target users include sales teams and managers: Responders (sales agents) who make calls, and Admins who oversee call quality and knowledge assets. The app’s key features include file upload (audio calls and company knowledge PDFs), automated call analysis (transcription, sentiment, compliance, and summary), interactive visualizations (charts, tables), and a real-time AI assistant. The business value lies in improving sales performance through data-driven insights and AI guidance.
•	Project purpose: Automate and augment sales call evaluation and coaching using AI.
•	Target users: Sales representatives/Dietitian (“Responders”) and their managers (“Admins”).
•	Core value: Real-time transcription and feedback improve agent effectiveness and customer engagement.

Instructions:
https://callsense0.vercel.app/
1. Create new user:

Home-> click Dietitian-> create profile(from bottom) -> enter name, email, password -> click Establish profile 
Now refresh the page and ask admin to approve your account.
Once your account is approved you can login

2. Upload call recordings:
 

Before uploading the recording fill above details. Client name and concern are required fields. Phone is optional

3. Admin-approve new user: 
Click on notification and find new users`
 

If user not available in notification then:
- go to  https://console.firebase.google.com/project/callsense-f8512/firestore/databases/-default-/data/~2Fusers
- find that user and change status to ‘VERIFIED’

4. Delete Reports:
Admin can delete all reports but dietitian can delete reports having conversation score less than 5. This is because sometimes due to downtime of gemini api, scores may be very low, near to zero.

5. Adding Instruction/ knowledge base:
Admins can add instruction for comparing calls and generating report.

 





6. To create new Admin:
-create a new account as dietitian
- go to https://console.firebase.google.com/project/callsense-f8512/firestore/databases/-default-/data/~2Fusers and find that user.
- edit and change role to ADMIN
 
Errors:
1.  this error occurs often:
“Model not available/Api quota Over”
Solution: On github Change gemini model name in: services/geminiService.ts
line: 39, 172, 258






Technology Stack
Category	Technology / Service	Role / Use
Frontend	React (TypeScript), Vite	UI framework and build tool
AI & ML	@google/genai (Gemini)	Call transcription, analysis, and live coaching (streaming)
Charts & Visualization	Recharts	Display of metrics and sentiment analysis
Markdown	Marked	Rendering analysis reports
Authentication	Firebase Auth	User sign-in, roles (ADMIN/RESPONDER)
Database	Firebase Firestore	Store users, call records, knowledge bases, feedback, settings
Storage	Firebase Storage	(If used) Store uploaded files (audio/PDF)
Deployment	Vite (static host) / Vercel	Builds static app; /api/gemini suggests a Vercel function
Icons/UI	lucide-react, React Type Animation	Icons and UI flair
Key libraries:
•	React 19 (via Vite) for UI components.
•	Google GenAI SDK (@google/genai) to call Gemini models for structured AI responses.
•	Firebase JS SDK for Auth and Firestore.
•	Recharts for charts (bar charts, etc.).
•	marked for rendering markdown reports.
•	Lucide icons for UI.
All configuration uses environment variables for secrets (e.g. process.env.GEMINI_API_KEY, Firebase keys) with sensible .env.local defaults in code. (Note: some Firebase keys are hard-coded defaults, which is a security risk as discussed below.)
Project Architecture
Callsense is structured as a frontend-centric, component-based app. Key directories:
•	src/ – main source code:
•	components/: React components (Dashboard, FileUpload, AnalysisReport, LiveCopilot, Logo).
•	services/: Data/API logic (firebaseService, geminiService, liveService, localDbService).
•	api/gemini.ts: A serverless API route (Vercel) for calling Google’s Gemini API from front-end.
•	utils/: Utility functions (e.g. leadAnalytics.ts for sentiment scoring).
•	Root files:
•	App.tsx: Main app, handles login and view switching.
•	index.tsx/index.html: App entry point.
•	package.json, tsconfig.json, vite.config.ts: Build configs.
•	firestore.rules: Security rules for Firestore (discussed in Security section).
•	README.md: Installation instructions.
flowchart LR
  subgraph Frontend
    App-->AuthService[Firebase Auth]
    App-->Dashboard
    App-->FileUpload
    App-->LiveCopilot
    App-->AnalysisReport
    Dashboard-->FirebaseService
    Dashboard-->GeminiAPI
    LiveCopilot-->LiveService
    FileUpload-->FirebaseService
  end
  subgraph Backend (Firebase/AI)
    FirebaseService-->Firestore[(Firestore DB)]
    FirebaseService-->Storage[(Storage)]
    GeminiAPI-->GoogleGenAI[Gemini API]
    LiveService-->GoogleGenAI
  end
  Frontend -- RealtimeAudio/WebSockets --> GeminiAPI
•	UI flow: Users sign up or log in (via Firebase Auth). The Dashboard component then allows uploading knowledge PDFs or call audio, initiating analyses. After processing, results are shown in AnalysisReport (transcripts, metrics, charts). The LiveCopilot component connects to Google’s Live API (websocket) to stream microphone audio and receive live suggestions.
•	Data flow: Uploaded files (PDF or audio) are first stored (local convert to Base64, then uploaded). The app calls geminiService to send the data to Gemini models. The AI returns structured JSON (via responseSchema) which is processed and saved in Firestore. The dashboard queries Firestore to display reports.
•	Deployment: The app is a static site (Vite build). The /api/gemini file suggests deployment on a serverless platform (e.g. Vercel functions) to avoid exposing API keys directly. The code imports Vercel types (VercelRequest) and uses @google/genai, so this route would run on Node. Firebase rules enforce access controls.
Features and User Journeys
Authentication Flow
•	Signup/Login: Users select role (Admin or Responder) and sign up with email/password (via firebaseService). New user entries are written to Firestore with status: 'UNVERIFIED'. Admins must verify new users (Firestore query for unverified, then change to VERIFIED).
Knowledge Management
•	Upload Knowledge (PDF): Admins upload company documentation (e.g. pricing, SOPs) as PDF. The app encodes the PDF to Base64 and calls geminiService.convertPdfToKnowledgeJson. This uses Gemini’s content API to extract structured company knowledge (e.g. rules, scripts, guidelines). The JSON is saved in Firestore as a KnowledgeBase entry (saveCompanyKnowledge), and Admin rules text may be stored in settings/companyKnowledge.
•	Use Case: The knowledge base informs later call analysis (to check pricing adherence and objection handling).
Call Analysis (File Upload)
•	Upload Call Recording: A Responder uploads an audio file of a sales call. The analyzeAudioCall function encodes audio to Base64 and sends it to Gemini’s Content API. The prompt instructs Gemini to “extract each conversation”, produce an English transcript array with timestamps, and identify the client’s name if mentioned. The code uses a response schema so Gemini outputs clean JSON rather than plain text【12†L224-L226】.
•	Second-stage AI Audit: The raw transcript is then fed back into Gemini with detailed instructions (sales auditing prompt), along with lead metadata (lead type and relationship) and company knowledge. The Gemini model evaluates objectives (e.g. price discussed, objections handled, buying signals) and outputs a structured CallAnalysis report (JSON) according to the CallAnalysis TypeScript interface. This includes metrics, compliance flags, transcript segments, and coaching feedback.
Call Analysis (Live Mode)
•	Live Copilot: The app can also listen to an on-going call via the user’s microphone. It uses the Gemini Live API (streaming WebSocket) for real-time analysis【19†L272-L280】. The code captures PCM audio from navigator.mediaDevices.getUserMedia, converts it to 16kHz PCM (base64), and streams it. The model listens and “whispers” suggestions back to the agent. The system instruction (“You are a Real-Time Sales Copilot… keep responses short and relevant to BATTLECARD”) guides the AI. Incoming messages trigger UI updates: transcript text (via onTranscript) and final suggestions (via onSuggestion). The Live API supports audio I/O; here, client-to-server streaming (per [19]) simplifies setup and reduces latency【19†L272-L280】.
Reporting and Dashboard
•	Metrics & Visualizations: After analysis, the app displays:
•	Transcript (with speaker labels and text).
•	Sentiment summary (positive/negative client sentiment, agent style).
•	Metrics: e.g. conversion score, politeness, listening scores.
•	Compliance Flags: booleans for objectives (e.g. dmVerified, priceAdherence).
•	Buying Signals Count and deal outcome.
•	Strengths/Weaknesses: textual feedback and actionable steps.
•	UI Components: The AnalysisReport component renders these, using Recharts for charts (bar charts for sentiment, etc.) and marked to format the textual summary. Admins can also view a list of past calls, filter by status, and export/download results.
Database Schema (ER Diagram)
The app primarily uses Firebase Firestore. Key collections and fields (see TypeScript interfaces for details):
•	users (user profiles)
•	id (UID), email, name, role (ADMIN or RESPONDER), status (VERIFIED/UNVERIFIED), avatar.
•	callAnalysis (call reports)
•	id, agentId (FK to users), agentName, clientName, clientPhone, clientConcern, duration, uploadDate, conversionScore, summary (markdown),
•	sentiment (client sentiment, agent style),
•	metrics (scores for politeness, listening, knowledge, etc.),
•	compliance (fields dmVerified, needDiscovery, priceAdherence, objectionHandled, hardCloseAttempted),
•	transcript (array of TranscriptSegment: speaker, timestamp, text, optional sentiment tagging),
•	feedback (strengths[], weaknesses[], actionableSteps[]),
•	deletedByResponder, createdAt.
•	knowledgeBases (company PDFs metadata)
•	id, fileName, url (Base64 data), mimeType, uploadedAt.
•	The JSON content (CompanyKnowledge) is likely stored in Firestore as well (fields: rules[], sops[], guidelines[], pricing, objectionHandling[], etc.).
•	settings (singleton docs)
•	e.g. settings/companyKnowledge doc holds rules text (admin-provided company policy).
•	responderFeedback (optional user comments)
•	id, responderId, feedbackText, createdAt.
 

Code Quality and Maintainability
•	TypeScript: The codebase uses TypeScript interfaces for data models (types.ts), aiding consistency. React components are function-based with hooks. Naming conventions are consistent (camelCase, descriptive names).
•	Folder Organization: Files are well-organized by feature. services/ abstracts data logic (Firebase calls, AI calls). components/ contains UI views.
•	Complexity: Some service functions are quite large (e.g. analyzeAudioCall ~400 lines). This could be refactored into smaller helper functions for readability. Error handling wraps AI calls but mostly rethrows errors. UI components handle async states and loading indicators appropriately.
•	DRY and Patterns: There is some code duplication or dead code:
•	localDbService methods are all deprecated no-ops (calls now go to Firebase).
•	Firestore converter patterns are used for data mapping.
•	The geminiService and liveService have custom logic; code comments could aid understanding.
•	Testing: No automated tests are included. Test coverage is 0% (no test/ folder). We recommend adding:
•	Unit tests for services (mock Firebase and AI calls).
•	Integration tests for user flows (login, upload, API calls).
•	Edge-case tests (invalid files, network failures).
•	Linting/Formatting: Standard tools (e.g. ESLint, Prettier) are not shown but could be added. The code uses standard React patterns and appears syntactically clean.
•	Documentation: The README is minimal (setup only). In-code JSDoc comments are sparse. We recommend documenting component props and service methods for future maintenance.
Security Analysis
•	Authentication/Authorization: Firebase Auth secures user identity. Firestore Security Rules enforce role-based access: for example, rules allow only Admins to verify users, and Responders can only delete their own reports under certain conditions. The rule set at firestore.rules is elaborate and uses custom functions (e.g. isVerified, isAdmin) to ensure role-based controls.
•	API Keys: The code uses process.env for sensitive keys (Gemini API key, Firebase config). However, default Firebase keys are present in code. These should never be committed (and indeed with Firebase, the API key is public-facing by design, but the rest of config).
•	Gemini API: The /api/gemini endpoint offloads the Google API key usage to the server. On the client side, geminiService and liveService call GoogleGenAI with process.env.API_KEY. Note: In a browser build, these env keys would be visible unless secured. The live example documentation explicitly recommends using ephemeral tokens instead of static keys for client streaming【19†L296-L299】. Using static API keys on the client is a vulnerability; we recommend switching to secure token auth for production.
•	Firestore Rules: Rules begin with rules_version = '2' and include checks like:
 	function isAuthenticated() { return request.auth != null; }
function isAdmin() { return isAuthenticated() && getUserData().role == "ADMIN"; }
function isResponder() { return isAuthenticated() && getUserData().role == "RESPONDER"; }
...
For instance, only Admins can delete any call report, while Responders can delete their own report only if its conversionScore < 10. This demonstrates careful access control. <br>Firebase Guidance: Official docs state that building user-based/role-based systems requires Firebase Auth + Firestore Rules【16†L1525-L1533】. The app’s rules follow this approach, binding request.auth.uid to data documents (e.g. verifying resource.data.agentId == request.auth.uid).
•	Input Validation: File uploads are sent as Base64 strings. The code trusts Gemini’s structured outputs; there is no client-side schema validation beyond parsing. We should validate inputs on server (e.g. file types). The analysisReport markdown is rendered with marked – if it included user content, an XSS risk might exist, but since it’s AI-generated content saved in Firestore, it’s presumably safe text.
•	Secrets Management: The .env.local is recommended for keys. However, ensure no real keys are in version control. Consider using Firebase Emulator or environment encryption for CI. Also avoid writing raw API keys in client bundles.
•	OWASP Considerations: The app does not expose user data insecurely. It uses HTTPS and Firebase’s secure channels by default. The main attack surface would be leaked keys or misconfigured rules. The Firestore rules disallow unauthorized reads/writes (e.g. only verified users can access, enforced in rules).
Recommendation: Remove hard-coded keys, enforce least privilege. For the Live API, use ephemeral token authentication instead of embedding the API key【19†L296-L299】 to reduce exposure.
Performance and Scalability
•	AI Calls: The most time-consuming operations are calls to Gemini models. Each PDF upload triggers one AI request to extract knowledge, and each audio call triggers two: one for transcription and one for auditing. Response times depend on model latency. The code sets temperature: 0.1 for determinism. Caching AI responses (e.g. storing transcription results) could avoid repeated calls for the same file.
•	WebSockets: The live copilot uses streaming WebSockets (AudioContext + Google Live API). This is efficient for real-time but can strain bandwidth. The app converts audio to 16-bit PCM, 16kHz (documented in [19]) to meet Gemini’s spec【19†L272-L280】. This keeps payload minimal. We should ensure the browser’s audio context handles 16000 Hz conversion correctly (the code uses audioContext.sampleRate).
•	Database: Firestore is a scalable NoSQL DB. The code queries with indexes (e.g. orderBy, limit). If call data grows large, ensure composite indexes for complex queries. For analytics charts, the dataset is small (per call), so client-side rendering is fine.
•	Caching: No explicit caching layer. Browser caching of static assets (Vite build) can be enabled. For API, the app could cache call reports in memory to reduce Firestore reads.
•	Bottlenecks: Uploading large files (audio, PDF) can be slow on weak connections. The app immediately base64-encodes files in-browser before sending. Consider showing progress indicators. Offloading heavy tasks (audio file to base64, PDF parsing) to Web Workers could keep UI responsive.
•	Concurrency: Firebase and Gemini APIs handle concurrency well. No stateful server to limit. If many live sessions run in parallel, API limits or cost could be factors (not discussed, but relevant for scaling).
AI/ML Components
Callsense heavily integrates Google’s Generative AI:
•	Gemini Content Models: The @google/genai SDK is used to call Gemini-2.5-flash models (text + base64 attachments). It generates structured JSON output by specifying responseSchema. Structured output is emphasized in Google’s docs as enabling “predictable, type-safe results” from unstructured input【12†L224-L226】.
•	PDF Understanding: The convertPdfToKnowledgeJson function feeds PDF data and a prompt like “Extract all company knowledge from this PDF…”, instructing Gemini to parse pricing, SOPs, FAQs, etc. The model yields a hierarchical JSON (CompanyKnowledge) of the PDF content.
•	Speech Recognition & Analysis: The analyzeAudioCall function first converts audio to text and then performs sales call auditing. Gemini’s multimodal capability (processing audio inline data) is used here. The prompt specifically asks for a detailed transcript and identifies the client’s name, outputting JSON. A second prompt audits sales performance (checking if agent met objectives, counted buying signals, etc.) and returns a JSON report matching the CallAnalysis schema. This leverages Gemini’s reasoning to evaluate compliance with sales best practices.
•	Live AI Copilot: For real-time calls, the app uses the Gemini Live API (streaming). The liveService sets up an audio WebSocket session. It sends audio frames as they’re captured, and receives text “suggestions” via server messages. The system message instructs Gemini to act as a sales copilot (see code: “Your goal: Help the Agent convert the lead. Listen for questions about pricing, features or objections…”). The live model gemini-2.5-flash-native-audio-preview-12-2025 is used. The technical spec indicates the model accepts raw PCM audio at 16kHz【19†L272-L280】. The Live API returns interim transcripts (input transcription) and final suggestions (output transcription). When the model’s turn completes, the code emits onSuggestion for the UI to display.
These AI components provide the core functionality: they replace manual transcription and analysis with automated intelligence. The repository suggests no other machine learning code—everything runs on Google’s hosted models. The developer uses prompt engineering (with detailed instructions and schema) to structure the output.
API Documentation
The project’s only explicit API endpoint is POST /api/gemini (a serverless function). Its behavior:
•	Endpoint: /api/gemini (relative to base URL)
•	Method: POST
•	Request JSON body:
 	{
  "model": "gemini-2.5-flash",
  "contents": { /* parts (inlineData or text) */ },
  "config": { /* optional settings */ }
}
•	Response:
•	200 on success with JSON { text: string }, where text is the AI’s response (string or JSON string).
•	400 if missing model or contents.
•	500 on failure (e.g. Gemini error).
•	Example: The function uses GoogleGenAI internally to forward the request to Gemini, then returns the .text of the response. Error structure is { error: "...", details: "..." }.
Authentication is assumed not needed at this endpoint (no auth checks in code), which is a risk: any client could call it with the API key. In production, the frontend would call this endpoint (without exposing the key). Without protection, a malicious user could abuse the endpoint for unlimited AI usage. We recommend implementing auth checks (e.g. verifying Firebase auth token) or using a proper proxy SDK.
The rest of the “API” is essentially the frontend code and Firebase:
•	Firebase SDK: The frontend calls Firebase via the services (firebaseService). These are not exposed as REST endpoints, but the app uses Firebase’s JS SDK directly to read/write data. For example, firebaseService.saveCompanyKnowledge() writes a document to callAnalysis or knowledgeBases. These operations are secured by Firestore Rules, not by our code.
•	Routes: No other custom HTTP routes. All logic is in-app and via Google/Firebase services.

