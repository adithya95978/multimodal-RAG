# Multimodal RAG API

This project implements a **Multimodal Retrieval-Augmented Generation (RAG)** API using **FastAPI**. It supports ingesting and querying both **text** and **image** data by leveraging **CLIP embeddings**, a vector database (**Pinecone** or in-memory fallback), and a multimodal Large Language Model (**Google Gemini**) for generative answers.

The application is structured to handle two types of knowledge bases:

  * **GKB (General Knowledge Base):** Publicly accessible data.
  * **SKB (Specific Knowledge Base):** User-specific and isolated data, keyed by the authenticated user's email.

-----

## 🚀 Key Features

  * **Multimodal Embeddings:** Uses the **CLIP** model (`openai/clip-vit-base-patch32`) to generate vector representations for both text and images.
  * **RAG Pipeline:** Orchestrates the flow from query embedding, vector search, content retrieval from S3, and final generation using **Gemini**.
  * **Document Processing:** Includes a service to extract **text and images from PDF** documents using PyMuPDF.
  * **Scalable Storage:** Uses **AWS S3** for persistent storage of original text and image files.
  * **Authentication:** Secured with **JWT** using OAuth2 Password Flow.

-----

## 📁 Project Structure

The codebase is organized as follows:

```
.
├── .env                 # Application configuration (not committed)
├── .gitignore
├── README.md
├── requirements.txt     # Python dependencies
└── app/
    ├── main.py          # FastAPI application entry point
    ├── api/
    │   └── v1/
    │       ├── auth.py      # Authentication (login, user details)
    │       ├── endpoints.py # Core API (query, embeddings, documents)
    │       └── schemas.py   # Pydantic data models
    ├── core/
    │   ├── config.py    # Application settings
    │   └── security.py  # Password hashing and JWT utilities
    ├── db/
    │   ├── __init__.py
    │   └── users.py     # Mock user database
    ├── scripts/
    │   └── bulk_ingest_gkb.py # CLI script for GKB ingestion
    └── services/
        ├── embedding.py      # CLIP embedding service
        ├── llm_gen.py        # Gemini generative service
        ├── parser.py         # PDF content extraction
        ├── rag_pipeline.py   # RAG workflow orchestrator
        ├── storage_service.py# AWS S3 file storage
        └── vector_db.py      # Pinecone/In-memory vector database
```

-----

## ⚙️ Setup and Installation

### Prerequisites

  * Python 3.9+
  * Access to:
      * **Google Gemini API** (for `GOOGLE_API_KEY`)
      * **Pinecone** (optional, an in-memory mock is used if not configured)
      * **AWS S3** (optional, for storage)

### Installation Steps

1.  **Clone the repository:**

    ```bash
    git clone <repository-url>
    cd multimodal-RAG
    ```

2.  **Create and activate a virtual environment:**

    ```bash
    python -m venv venv
    source venv/bin/activate  # On Linux/macOS
    # .\venv\Scripts\activate # On Windows
    ```

3.  **Install dependencies:**

    ```bash
    pip install -r requirements.txt
    ```

    *The dependencies include `pinecone`, `transformers`, `torch`, `google-generativeai`, `fastapi`, `pydantic-settings`, and `boto3`*.

4.  **Configure Environment Variables**

    Create a file named `.env` in the root directory and populate it with your credentials. Refer to `app/core/config.py` for required variables:

    ```ini
    # .env file

    # Google Generative AI
    GOOGLE_API_KEY="YOUR_GOOGLE_API_KEY"

    # Pinecone (Optional - In-memory adapter used if keys are missing)
    PINECONE_API_KEY="YOUR_PINECONE_API_KEY"
    PINECONE_ENVIRONMENT="YOUR_PINECONE_ENVIRONMENT"
    # PINECONE_INDEX_NAME defaults to "multi-rag-index"

    # AWS S3 (Optional - Storage Service disabled if keys are missing)
    AWS_ACCESS_KEY_ID="YOUR_AWS_ACCESS_KEY_ID"
    AWS_SECRET_ACCESS_KEY="YOUR_AWS_SECRET_ACCESS_KEY"
    AWS_REGION="YOUR_AWS_REGION"
    S3_BUCKET_NAME="YOUR_S3_BUCKET_NAME"

    # Authentication (Change SECRET_KEY in production!)
    SECRET_KEY="a_very_secret_key_that_should_be_changed"
    ```

5.  **Run the application:**

    ```bash
    uvicorn app.main:app --reload
    ```

    The API will be available at `http://127.0.0.1:8000`.

-----

## 🛠 Usage and Endpoints

The API documentation (Swagger UI) is available at `http://127.0.0.1:8000/docs`.

### Authentication

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/v1/auth/token` | Generates a JWT **access token** upon successful username/password (**`form_data.username`** is the email, e.g., `test@example.com`). |
| `GET` | `/api/v1/auth/users/me/` | Retrieves the details of the **current active user**. |

### Core API

All core endpoints (except health check) require a valid JWT `Authorization: Bearer <token>` header.

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/v1/` | **Health Check** and displays a welcoming message. |
| `POST`| `/api/v1/embeddings/` | Creates a vector embedding for the provided **`text`** or **`image`** and upserts it to the **GKB** or **SKB**. |
| `POST`| `/api/v1/documents/extract/` | Extracts text and images from an uploaded **PDF** file. |
| `POST`| `/api/v1/query/` | Performs the full **RAG pipeline** by finding relevant context (text/image) based on the **`query`** and generating an answer. |

### Bulk Ingestion Script

The `bulk_ingest_gkb.py` script is a command-line utility for mass ingestion of PDF files into the **GKB** (General Knowledge Base).

1.  **Run the script:**
    ```bash
    python app/scripts/bulk_ingest_gkb.py <path_to_pdf_or_directory>
    ```
2.  The script will:
      * Extract text and images from each PDF.
      * Upload the original content to **AWS S3**.
      * Create and upsert embeddings for both the text and images into the vector database.
