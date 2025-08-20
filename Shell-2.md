```python

from typing import Optional, Literal
from sqlmodel import SQLModel, Field, Relationship

ComponentType = Literal["header", "footer", "navigation"]

class Media(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    filename: str
    url: str

class Component(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    ctype: ComponentType
    raw_html: str  # contains {{vars}}
    compiled_html: str  # media vars replaced to real URLs

class Shell(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    header_id: int
    footer_id: int
    navigation_id: int
    raw_html: str  # contains {{header}}, {{footer}}, {{navigation}}, {{content}}
    compiled_html: str  # components injected, keeps {{content}} for event stage

class Event(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    shell_id: int
    content_html: str
    final_html: str

import re
from typing import Dict

VAR_RE = re.compile(r"{{\s*([a-zA-Z_][a-zA-Z0-9_])\s}}")

def render_template(template: str, context: Dict[str, str]) -> str:
    def repl(match):
        key = match.group(1)
        return str(context.get(key, match.group(0)))  # leave untouched if missing
    return VAR_RE.sub(repl, template)

from fastapi import FastAPI, UploadFile, File, Form, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from sqlmodel import select, Session
from pathlib import Path
from typing import Dict, Literal

# Assumed imports from database.py and utils.py
# from database import init_db, get_session
# from models import Media, Component, Shell, Event, ComponentType
# from utils import render_template

app = FastAPI(title="Shell System API")

# CORS (adjust origins as needed)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Static files for uploaded media
UPLOAD_DIR = Path(__file__).parent / "uploads"
UPLOAD_DIR.mkdir(parents=True, exist_ok=True)
app.mount("/uploads", StaticFiles(directory=UPLOAD_DIR), name="uploads")

@app.on_event("startup")
def on_startup():
    # init_db()
    pass

# ---- Media Endpoints ------------------------------------------------------

@app.post("/media", response_model=Media)
async def upload_media(file: UploadFile = File(...), session: Session = Depends(lambda: None)):
    dest = UPLOAD_DIR / file.filename
    with dest.open("wb") as f:
        f.write(await file.read())
    media = Media(filename=file.filename, url=f"/uploads/{file.filename}")
    # session.add(media)
    # session.commit()
    # session.refresh(media)
    return media

@app.get("/media", response_model=list[Media])
def list_media(session: Session = Depends(lambda: None)):
    # return session.exec(select(Media)).all()
    return []

# ---- Component Endpoints (header/footer/navigation) -----------------------

class ComponentCreatePayload(dict):
    name: str
    ctype: ComponentType
    raw_html: str
    media_map: Dict[str, int]  # e.g. {"logo": 1, "banner": 2}

@app.post("/components", response_model=Component)
async def create_component(
    name: str = Form(...),
    ctype: ComponentType = Form(...),
    raw_html: str = Form(...),
    media_map: str = Form("{}"),  # JSON string {var: mediaId}
    session: Session = Depends(lambda: None),
):
    import json
    try:
        var_map: Dict[str, int] = json.loads(media_map or "{}")
    except Exception:
        raise HTTPException(status_code=400, detail="media_map must be JSON")

    # Build context from media IDs → URLs
    context: Dict[str, str] = {}
    for var, mid in var_map.items():
        # m = session.get(Media, mid)
        # if not m:
        #     raise HTTPException(status_code=400, detail=f"Media id {mid} not found for var '{var}'")
        # context[var] = m.url
        context[var] = f"/uploads/media_{mid}.jpg"  # Mock URL
    
    compiled = render_template(raw_html, context)
    comp = Component(name=name, ctype=ctype, raw_html=raw_html, compiled_html=compiled)
    # session.add(comp)
    # session.commit()
    # session.refresh(comp)
    return comp

@app.get("/components", response_model=list[Component])
async def list_components(session: Session = Depends(lambda: None)):
    # return session.exec(select(Component)).all()
    return []

# ---- Shell Endpoints ------------------------------------------------------

@app.post("/shells", response_model=Shell)
async def create_shell(
    name: str = Form(...),
    header_id: int = Form(...),
    footer_id: int = Form(...),
    navigation_id: int = Form(...),
    raw_html: str = Form(...),  # contains {{header}}, {{footer}}, {{navigation}}, {{content}}
    session: Session = Depends(lambda: None),
):
    # header = session.get(Component, header_id)
    # footer = session.get(Component, footer_id)
    # nav = session.get(Component, navigation_id)
    # if not header or not footer or not nav:
    #     raise HTTPException(status_code=400, detail="Invalid component id(s)")
    
    # Mock components for demonstration
    header_html = f"<div>Header from component {header_id}</div>"
    footer_html = f"<div>Footer from component {footer_id}</div>"
    nav_html = f"<div>Navigation from component {navigation_id}</div>"

    compiled = render_template(
        raw_html,
        {
            "header": header_html,
            "footer": footer_html,
            "navigation": nav_html,
        },
    )  # Keep {{content}} for later event rendering.
    
    shell = Shell(
        name=name,
        header_id=header_id,
        footer_id=footer_id,
        navigation_id=navigation_id,
        raw_html=raw_html,
        compiled_html=compiled,
    )
    # session.add(shell)
    # session.commit()
    # session.refresh(shell)
    return shell

@app.get("/shells", response_model=list[Shell])
async def list_shells(session: Session = Depends(lambda: None)):
    # return session.exec(select(Shell)).all()
    return []

# ---- Event Endpoints ------------------------------------------------------

@app.post("/events", response_model=Event)
async def create_event(
    name: str = Form(...),
    shell_id: int = Form(...),
    content_html: str = Form(""),  # inserted into {{content}}
    session: Session = Depends(lambda: None),
):
    # shell = session.get(Shell, shell_id)
    # if not shell:
    #     raise HTTPException(status_code=400, detail="Shell not found")
    
    # Mock shell for demonstration
    mock_compiled_shell = f"<html><body><div>Mock shell {shell_id}</div>{{{{content}}}}</body></html>"

    final_html = render_template(mock_compiled_shell, {"content": content_html})
    event = Event(name=name, shell_id=shell_id, content_html=content_html, final_html=final_html)
    # session.add(event)
    # session.commit()
    # session.refresh(event)
    return event

@app.get("/events", response_model=list[Event])
async def list_events(session: Session = Depends(lambda: None)):
    # return session.exec(select(Event)).all()
    return []

@app.get("/events/{event_id}/html")
async def get_event_html(event_id: int, session: Session = Depends(lambda: None)):
    # ev = session.get(Event, event_id)
    # if not ev:
    #     raise HTTPException(status_code=404, detail="Event not found")
    # return {"html": ev.final_html}
    return {"html": f"<html><body><h1>Final HTML for event {event_id}</h1><p>Mock content</p></body></html>"}
```python
