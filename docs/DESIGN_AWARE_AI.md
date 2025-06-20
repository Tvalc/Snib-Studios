# Design Document: Design-Aware AI Assistant

## 1. Overview

### 1.1. Problem Statement
The current AI Assistant in the AI Studio often rewrites entire game files when asked to make updates, rather than performing modular, targeted edits. This happens because the AI lacks sufficient context about the project's specific architecture, existing code structure, and design principles. It defaults to generating new, complete code based on general patterns, which overwrites carefully crafted work.

### 1.2. Proposed Solution
To solve this, we will evolve the assistant into a **Design-Aware AI**. This system will integrate project-specific design documents directly into the AI's workflow. Before processing a user's request, a new "Context Engine" will retrieve relevant information from the codebase and a dedicated Design Document Store. It will then assemble a comprehensive "mega-prompt" that guides the AI to make precise, context-aware edits that respect the project's established architecture and rules.

## 2. System Architecture & Design

The proposed system consists of three core components: a centralized store for design documents, a backend "Context Engine" to build intelligent prompts, and a structured "Mega-Prompt" to guide the AI.

### 2.1. Centralized Design Document Store

This is the "source of truth" for each project's design.

*   **Format**: **Markdown (`.md`)**. This format is both human-readable and easy for our services to parse.
*   **Location**: A new `design/` directory will be created at the root of each user's game project.
*   **Structure**: The `design/` directory will contain a standardized set of documents:
    *   `architecture.md`: Defines high-level code structure, state management patterns, and component responsibilities.
    *   `gameplay_mechanics.md`: Details core loops, character abilities, scoring, rules, and win/loss conditions.
    *   `art_and_style.md`: Outlines visual themes, color palettes, UI style guides, and sound design notes.
    *   `technical_specs.md`: Specifies target platforms, performance constraints, and required external APIs.
*   **Storage & Versioning**: The documents will be stored in the project's **Git repository**. This leverages existing GitHub integration for version control, collaboration, and history tracking.

### 2.2. The Context Engine Service

This new backend service is the brain of the system. It intercepts user requests and enriches them with deep context before they reach the LLM.

**Responsibilities:**

1.  **Request Analysis**: Parses the user's natural language request to identify key intents and entities.
2.  **Design Doc Vectorization**:
    *   On project load, or when files in the `design/` directory are updated, the engine will parse the Markdown documents into logical chunks (e.g., a paragraph, a section).
    *   These chunks will be converted into **vector embeddings** using a sentence-transformer model.
    *   The embeddings will be stored in a **Vector Database**. `pgvector`, a PostgreSQL extension, is the recommended choice as it integrates seamlessly with our existing Supabase stack.
3.  **Contextual Retrieval**:
    *   When a user submits a request, the engine converts the request into a vector.
    *   It performs a similarity search against the vector database to find the most relevant chunks from the design documents.
    *   Simultaneously, it performs a search across the codebase (e.g., using semantic search or file-based heuristics) to find the most relevant code snippets.
4.  **Prompt Assembly**: The engine constructs a "Mega-Prompt" by combining the retrieved context into a single, comprehensive instruction for the AI.

### 2.3. The "Mega-Prompt" Structure

This is the final package sent to the LLM. It's designed to give the AI all the information it needs to make an intelligent, surgical edit.

```
[System Instruction]
You are an expert game developer. Your task is to EDIT the provided code to meet the user's request.
You MUST adhere to the principles outlined in the attached Design Document snippets.
Do NOT rewrite entire files. Make precise, targeted changes to the existing code.

[User Request]
"{{user_natural_language_request}}"

[Design Document Context]
(This section contains the top 3-5 most relevant chunks retrieved from the vector database.)
File: {{source_document_name.md}}
"{{retrieved_design_doc_chunk_1}}"

File: {{source_document_name.md}}
"{{retrieved_design_doc_chunk_2}}"

[Relevant Code Snippets]
(This section contains the most relevant code snippets from the user's project.)
File: {{source_code_file_1.js}}
```
{{retrieved_code_snippet_1}}
```

File: {{source_code_file_2.js}}
```
{{retrieved_code_snippet_2}}
```

[Final Instruction for the AI]
Based on all the information above, provide the necessary code modifications to implement the user's request.
```

## 3. Workflow Diagram

The end-to-end data flow is visualized below.

```mermaid
graph TD
    subgraph "User Interface"
        A[User in AI Studio] -- "User's Request" --> B{AI Assistant UI}
    end

    subgraph "Backend: Context Engine Service"
        C[1. Request Analysis]
        D{2. Vector Database<br>(pgvector on Supabase)}
        E[3. Codebase Access]
        F[4. Prompt Assembler]
    end
    
    subgraph "Backend: AI Service (LLM)"
        G[Large Language Model]
    end

    B -- Sends Request --> C
    C -- Vectorized Query --> D
    D -- Returns Relevant Design Chunks --> F
    C -- Semantic Search Query --> E
    E -- Returns Relevant Code Snippets --> F
    F -- Assembles "Mega-Prompt" --> G
    G -- Returns Targeted Code Edit --> B
```

## 4. High-Level Implementation Plan

This project will be broken down into distinct phases to ensure a robust and iterative rollout.

1.  **Phase 1: Foundation & Schemas**
    *   **Task**: Formally define and document the schema for the `design/` directory and its files.
    *   **Task**: Set up `pgvector` on our Supabase instance and create the necessary tables for storing embeddings.
    *   **Task**: Develop the initial service for parsing, chunking, and vectorizing the design documents.
    *   **Goal**: Ability to convert design docs into a searchable vector store.

2.  **Phase 2: Context Engine & Prompt Assembly**
    *   **Task**: Build the backend logic for the Context Engine. This includes the API endpoint that receives user requests.
    *   **Task**: Implement the vector search functionality to retrieve design context.
    *   **Task**: Implement the code retrieval functionality.
    *   **Task**: Build the Prompt Assembler logic to create the final "Mega-Prompt".
    *   **Goal**: A service that can take a user request and output a complete, context-rich prompt.

3.  **Phase 3: Integration & UI**
    *   **Task**: Integrate the Context Engine with the existing AI Assistant frontend.
    *   **Task**: Update the UI to provide transparency to the user. For example, display a "Referencing: `gameplay_mechanics.md`" message so the user knows the AI is using the documents.
    *   **Task**: Implement end-to-end testing of the new flow.
    *   **Goal**: A fully functional, design-aware AI assistant operating within the AI Studio.

4.  **Phase 4: Tooling & Refinement**
    *   **Task**: Build UI tools within the Studio to make editing the design documents easier for users.
    *   **Task**: Monitor the performance and accuracy of the system and refine the prompting and retrieval strategies.
    *   **Goal**: A polished, user-friendly system that is easy to maintain and improve.

## 5. User Interface and Interaction (UI/UX)

To ensure the system is powerful yet intuitive, the user interface will be updated to seamlessly support this new, design-aware workflow.

### 5.1. Integrated Design Doc Editor
*   **Component**: A dedicated **"Design" tab** will be added to the main AI Studio interface.
*   **Functionality**: This tab will feature a file explorer for the `/design` directory and a rich Markdown editor with a live preview. This allows users to create and manage their design documents without leaving the platform.

### 5.2. AI-Assisted Document Creation
*   **Feature**: The "Design" tab will include an **"AI Bootstrap"** feature.
*   **Workflow**: A user can provide a high-level concept (e.g., "Create a design for an enemy guard with patrol behavior"). The AI will then generate a new, well-structured Markdown file (e.g., `design/enemies/guard.md`) pre-populated with relevant sections, adhering to our standardized templates.

### 5.3. Contextual Transparency in AI Chat
*   **Goal**: Build user trust and provide clear debugging information.
*   **Implementation**: When the AI generates a response based on a design document, it will explicitly cite its source at the top of the response.
*   **Example UI Message**: `> Referencing design/gameplay_mechanics.md...`
*   **Benefit**: This immediately informs the user whether an incorrect output stems from a flawed design document or a misinterpretation by the AI.

### 5.4. Conflict Resolution Workflow
*   **Rule**: The AI must not blindly execute a request that contradicts a design document. It must flag the conflict and request user clarification.
*   **Example AI Response**: "Your request to set player health to 500 contradicts `gameplay_mechanics.md`, which states the maximum is 100. How would you like to proceed?"
*   **User Options**: The user will be presented with clear choices, such as:
    1.  **[Update Design Doc]**: Approves the change and queues an automated update to the relevant `.md` file.
    2.  **[Revise Request]**: Allows the user to rephrase their request to be compliant.
    3.  **[Ignore for Now]**: Proceeds with the code change as a one-time exception.

## 6. AI Feedback and Learning Loop

To ensure the AI's performance improves over time, we will implement a continuous feedback and learning loop.

### 6.1. User Feedback Mechanism
*   **UI**: Simple **Thumbs Up (`👍`)** and **Thumbs Down (`👎`)** icons will appear next to every significant code modification generated by the AI.
*   **Data Collection**: An optional text field will allow users to provide specific reasons for their ratings, especially for negative feedback.

### 6.2. Feedback Data Storage
*   **Backend**: All feedback events will be logged in a dedicated table within our Supabase database.
*   **Schema**: The log entry will capture the complete context: the initial user request, the full "Mega-Prompt" sent to the LLM, the AI's generated output, the user's rating, and any text comments.

### 6.3. Continuous Improvement Strategy
*   **Short-Term (Prompt Engineering)**: We will regularly analyze the logs of negatively-rated interactions. This data will provide immediate, actionable insights for refining our prompt templates within the Context Engine.
*   **Long-Term (Model Fine-Tuning)**: The collected feedback data will serve as a high-quality dataset for fine-tuning a smaller, specialized language model in the future. A fine-tuned model will offer higher accuracy for our specific domain and be more cost-effective at scale.

## 7. Error Handling and Graceful Fallbacks

The system must be resilient and handle scenarios where the ideal context is unavailable.

### 7.1. Missing Design Documents
*   **Condition**: If a project's `/design` directory is empty or does not exist.
*   **Behavior**: The Context Engine will gracefully fall back to its original behavior, using only the available code context.
*   **UI Notification**: The Studio will display a helpful, non-blocking notification: "For more accurate AI assistance, create your project's design documents in the 'Design' tab."

### 7.2. Irrelevant Context
*   **Condition**: If the vector search for design documents returns no results with a similarity score above a predefined threshold.
*   **Behavior**: The AI will proceed with the request but will inform the user that it is operating without specific design guidance.
*   **Example AI Response**: "I couldn't find any specific design guidance for your request. I will proceed based on my general knowledge, but the result may not align with your project's architecture." 