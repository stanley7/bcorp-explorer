# bcorp-explorer

# B Corp Explorer

A retrieval-augmented chat assistant for exploring B Corp certified companies. It answers questions like "which companies are in Canada?" or "what's this company's governance score?" by retrieving relevant company profiles from a vector database and passing them to an LLM for a grounded answer.

## What it is about

The project has two parts:

**Backend** - a Retrieval-Augmented Generation (RAG) pipeline that runs in Google Colab. It embeds a dataset of B Corp company profiles into a Chroma vector store, retrieves the most relevant profiles for a given question, and passes them to Groq's Llama 3.3 70B model to generate an answer. It's exposed to the web as a FastAPI server, tunneled through ngrok, and remembers recent conversation turns so follow-up questions work.

**Frontend** - a standalone HTML/CSS/JS chat interface (no build step, no framework) that talks to the backend over a simple `/chat` API. It can be opened directly in a browser or deployed as a static site (e.g. on Vercel).

```
[ Browser / Vercel-hosted HTML ]
            |
            |  fetch(API_URL + "/chat")
            v
     [ ngrok tunnel ]
            |
            v
[ FastAPI server running in Colab ]
            |
            v
   [ RAG chain: Chroma retriever -> Groq LLM ]
```

## Dataset: extraction and location

The chat assistant answers questions using a single JSON file, `bcorp_extracted.json`, containing one record per B Corp company. Each record includes fields like:

- `company_name`
- `headquarters`
- `certified_since`
- `industry`
- `sector`
- `operates_in` (list of countries/regions)
- `website`
- `b_impact_score`
- `governance_score`

**Location:** the notebook expects this file in Google Drive at:

```
/content/drive/MyDrive/bcorp_extracted.json
```

If your file lives somewhere else, update the `json_path` variable in cell 2 of the backend script to match. The notebook loads this JSON, wraps each company record in a LangChain `Document`, and embeds it into the vector store, it does not scrape or extract the data itself, so you'll need `bcorp_extracted.json` already prepared before running the backend.

## Setup

### 1. Backend (Google Colab)

1. Open a new Colab notebook and paste in the backend script cell by cell, in order.
2. **Cell 1** - install dependencies. Uncomment the `!pip install` line and run it.
3. **Add your Groq API key as a Colab secret** — click the key icon in the left sidebar, add a new secret named `groq`, and paste your Groq API key as the value. (Get a key at [console.groq.com](https://console.groq.com).)
4. **Cell 2** - mounts your Google Drive and loads `bcorp_extracted.json`. Approve the Drive access prompt when it appears.
5. **Cell 3** - builds the Chroma vector store from the loaded documents. This can take a minute or two depending on dataset size.
6. **Cell 4** - builds the RAG chain (retriever + Groq LLM, with chat-history-aware retrieval for follow-up questions).
7. **Cell 5** - starts the FastAPI server and opens an ngrok tunnel.
   - Before running, replace `"YOUR_NGROK_AUTHTOKEN"` with your own token from [dashboard.ngrok.com/get-started/your-authtoken](https://dashboard.ngrok.com/get-started/your-authtoken).
   - Once it runs, it prints a public URL like `https://xxxx.ngrok-free.app` — copy this.

### 2. Frontend

1. Open the HTML chat file in a text editor.
2. Near the top of the `<script>` tag, find:
```js
   const API_URL = 'PASTE_YOUR_TUNNEL_URL_HERE';
```
3. Replace the placeholder with the ngrok URL from backend step 7.
4. Save the file. Open it directly in a browser to test, or deploy it as a static site (e.g. drag-and-drop onto [vercel.com/new](https://vercel.com/new), or connect this repo to Vercel).

### 3. Keeping it running

The backend only works while the Colab notebook is actively connected and running:

- Free Colab sessions disconnect after ~90 minutes idle, and are capped at ~12 hours regardless of activity.
- Every time you restart the notebook, ngrok issues a **new** URL — repeat backend step 7 and update `API_URL` in the frontend again.
- For something that needs to stay reachable without babysitting a notebook, move the FastAPI backend to a persistent host (Render, Railway, Fly.io, etc.) instead of Colab + ngrok.

## Notes / limitations

- Chat memory is capped at the last 10 messages per conversation to keep token usage bounded.
- Free ngrok tunnels show visitors an interstitial warning page by default; the frontend already sends the `ngrok-skip-browser-warning` header to skip it.
- This setup is meant for demos and prototyping, not production traffic.
