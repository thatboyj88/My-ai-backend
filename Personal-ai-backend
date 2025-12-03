# main.py - minimal FastAPI app with Playwright scraping endpoint
from fastapi import FastAPI, HTTPException, Request, Header
from pydantic import BaseModel
import os, time, logging

app = FastAPI(title="Personal AI Backend Minimal")

# simple auth check for MVP
def check_auth(authorization: str | None = Header(default=None)):
    if not authorization:
        raise HTTPException(status_code=401, detail="Missing Authorization header")
    if authorization.startswith("Bearer "):
        token = authorization.split(" ",1)[1]
    else:
        token = authorization
    # Accept demo-token or env JWT_SECRET for MVP
    if token in ("demo-token", os.getenv("JWT_SECRET","devsecret")):
        return True
    raise HTTPException(status_code=401, detail="Invalid token")

class ChatReq(BaseModel):
    question: str
    url: str | None = None
    task: str | None = None  # optional task like "screenshot"

@app.get("/health")
async def health():
    return {"status":"ok"}

@app.post("/chat")
async def chat(req: ChatReq, authorization: str | None = Header(default=None)):
    check_auth(authorization)
    # Very small behavior:
    # - if url supplied, try to render page using Playwright (if available)
    # - otherwise return a dummy response or ask to set OPENAI_API_KEY
    if req.url:
        # attempt to render with Playwright if installed and playable on host
        try:
            from playwright.sync_api import sync_playwright
            with sync_playwright() as p:
                browser = p.chromium.launch(headless=True, args=["--no-sandbox"])
                page = browser.new_page()
                page.goto(req.url, timeout=30000)
                time.sleep(2)
                html = page.content()
                browser.close()
            # quick cleaning: strip long whitespace
            text = " ".join(html.split())[:4000]
            return {"answer": f"Fetched and rendered {req.url}. First 4000 chars: {text[:800]}...", "source": req.url}
        except Exception as e:
            # Playwright might not be available on some hosts; return fallback
            return {"answer": f"Could not render with Playwright: {str(e)}. If you just want LLM responses, set OPENAI_API_KEY and ask without url.", "error": str(e)}
    else:
        # No URL — use OpenAI if key present
        OPENAI_KEY = os.getenv("OPENAI_API_KEY")
        if not OPENAI_KEY:
            return {"answer": "OPENAI_API_KEY not configured on backend. Set it in your render environment variables."}
        # minimal OpenAI chat reply using REST (avoid heavy deps)
        try:
            import requests, json
            headers = {"Authorization": f"Bearer {OPENAI_KEY}", "Content-Type":"application/json"}
            payload = {
                "model": "gpt-4o-mini",
                "messages": [{"role":"user","content": req.question}],
                "max_tokens": 400
            }
            r = requests.post("https://api.openai.com/v1/chat/completions", headers=headers, data=json.dumps(payload), timeout=30)
            j = r.json()
            text = j["choices"][0]["message"]["content"]
            return {"answer": text}
        except Exception as e:
            return {"answer": f"OpenAI request failed: {str(e)}"}
