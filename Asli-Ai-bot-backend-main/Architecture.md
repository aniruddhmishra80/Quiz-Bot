# QuizGenX System Architecture & Data Flow

## 1. Tech Stack Overview

### Frontend
- **Framework**: React 19 / Vite
- **Routing**: React Router DOM (v7)
- **Styling**: Vanilla CSS (Custom design variables, Glassmorphism, Dark Mode)
- **HTTP Client**: Axios
- **Markdown & Math**: `react-markdown`, `rehype-katex`, `remark-math` (for mathematical formulation rendering)

### Backend
- **Server**: Node.js / Express 5
- **AI / LLMs**: Google Gemini 1.5/2.0 (`@google/generative-ai`) and LangChain Google integration (`@langchain/google-genai`).
- **File Processing**: 
  - `multer` (multipart requests)
  - `pdf-parse` (PDF extraction)
  - `mammoth` (Word document extraction `.docx`)
- **Memory & Embeddings**: Custom in-memory vectors paired with Gemini Text Embedding endpoints to support Retrieval-Augmented Generation (RAG).

---

## 2. System Architecture

QuizGenX employs a classic Client-Server Architecture decoupled for scale.

- **Client (React UI)**: Handles the user interface, tracks conversation history, and dynamically conditionally renders markdown chat blocks or interactive `QuizComponent` interfaces based on API payloads.
- **Node/Express API**: A lightweight routing backend that orchestrates file parsing, orchestrating the LLM text completion, and embedding computation for semantic similarity searches.
- **LLM Engine**: External API dependency (Google Gemini). Serves both as the **Embedding Engine** mapping file fragments to vectors, and the **Generation Engine** building structured Quizzes and conversational replies according to the System Prompts.

---

## 3. Data Flow

### Flow A: File Upload & Indexing (RAG Intake)
1. **Upload Trigger**: User selects a PDF/Word file in the frontend UI.
2. **Buffer Stream**: File is posted via multipart form-data to `POST /upload`. 
3. **Extraction**: `multer` intercepts the file in-memory. Based on MIME type, it is handed to `pdf-parse` or `mammoth` to extract pure text blocks.
4. **Chunking**: The extracted text is chopped into manageable character blocks (`chunkText.js`) with slight overlaps to preserve sentence boundaries.
5. **Embedding**: `embeddingService.js` batches the chunks and sends them to the Gemini Embedding model. Vector arrays are returned and saved locally paired with the text chunk.

### Flow B: Quiz Generation & Chat Request (RAG Output)
1. **User Query**: User clicks the "Generate a quiz..." chip or types a custom topic.
2. **Similarity Search**: `ask.js` intercepts the query. The user query is itself embedded. The system then compares the query vector to the stored chunk vectors using **Cosine Similarity**, surfacing the Top K most relevant textual blocks from the document.
3. **Prompt Composition**: The backend builds a payload inserting the retrieved chunks into a specific `SYSTEM_PROMPT` instructing Gemini to output strict JSON multiple-choice questions.
4. **LLM Generation**: LangChain invokes Gemini. The model analyzes the context restrictions and streams back the generated JSON.
5. **Frontend Interpretation**: `ChatView.jsx` runs robust regex/JSON interceptors. 
   - If the content parses as the defined Quiz Object Array, the UI delegates rendering to `<QuizComponent />`.
   - If not, it parses to `<ChatMarkdown />`.
6. **Interaction**: The user interacts with the generated Quiz buttons; the frontend evaluates boolean values directly computed against the payload's `answer` strings.
