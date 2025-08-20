# Shell-management-

Full-Stack Implementation: Reusable Shell System with React (Vite + TS) & FastAPI (SQLModel + SQLite)

This project lets users:

1. Upload media files.


2. Create Header/Footer/Navigation components with HTML templates that reference media via {{vars}}, and compile them to HTML with media URLs.


3. Create a Shell by selecting a header/footer/navigation and writing a layout with {{header}}, {{footer}}, {{navigation}}, and {{content}}. The shell compiles components into the layout and keeps {{content}} for later.


4. Create an Event by selecting a shell and entering content to produce a final page.



Below is a minimal but complete setup you can paste into a new repo.


---

Backend — FastAPI

Structure

backend/
  main.py
  database.py
  models.py
  utils.py
  requirements.txt
  uploads/            # created at runtime, stores uploaded files

backend/requirements.txt

fastapi==0.115.0
uvicorn==0.30.6
sqlmodel==0.0.21
pydantic==2.9.2
python-multipart==0.0.9
jinja2==3.1.4

backend/database.py

from sqlmodel import SQLModel, create_engine, Session
from pathlib import Path

DB_PATH = Path(__file__).parent / "app.db"
engine = create_engine(f"sqlite:///{DB_PATH}", connect_args={"check_same_thread": False})


def init_db():
    SQLModel.metadata.create_all(engine)


def get_session():
    with Session(engine) as session:
        yield session

backend/models.py

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

backend/utils.py

import re
from typing import Dict

VAR_RE = re.compile(r"\{\{\s*([a-zA-Z_][a-zA-Z0-9_]*)\s*\}\}")


def render_template(template: str, context: Dict[str, str]) -> str:
    def repl(match):
        key = match.group(1)
        return str(context.get(key, match.group(0)))  # leave untouched if missing
    return VAR_RE.sub(repl, template)

backend/main.py

from fastapi import FastAPI, UploadFile, File, Form, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from sqlmodel import select, Session
from pathlib import Path
from typing import Dict, Literal

from database import init_db, get_session
from models import Media, Component, Shell, Event, ComponentType
from utils import render_template

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
    init_db()


# ---- Media Endpoints ------------------------------------------------------
@app.post("/media", response_model=Media)
async def upload_media(file: UploadFile = File(...), session: Session = Depends(get_session)):
    dest = UPLOAD_DIR / file.filename
    with dest.open("wb") as f:
        f.write(await file.read())
    media = Media(filename=file.filename, url=f"/uploads/{file.filename}")
    session.add(media)
    session.commit()
    session.refresh(media)
    return media

@app.get("/media", response_model=list[Media])
def list_media(session: Session = Depends(get_session)):
    return session.exec(select(Media)).all()


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
    session: Session = Depends(get_session),
):
    import json
    try:
        var_map: Dict[str, int] = json.loads(media_map or "{}")
    except Exception:
        raise HTTPException(status_code=400, detail="media_map must be JSON")

    # Build context from media IDs → URLs
    context: Dict[str, str] = {}
    for var, mid in var_map.items():
        m = session.get(Media, mid)
        if not m:
            raise HTTPException(status_code=400, detail=f"Media id {mid} not found for var '{var}'")
        context[var] = m.url

    compiled = render_template(raw_html, context)
    comp = Component(name=name, ctype=ctype, raw_html=raw_html, compiled_html=compiled)
    session.add(comp)
    session.commit()
    session.refresh(comp)
    return comp

@app.get("/components", response_model=list[Component])
async def list_components(session: Session = Depends(get_session)):
    return session.exec(select(Component)).all()


# ---- Shell Endpoints ------------------------------------------------------
@app.post("/shells", response_model=Shell)
async def create_shell(
    name: str = Form(...),
    header_id: int = Form(...),
    footer_id: int = Form(...),
    navigation_id: int = Form(...),
    raw_html: str = Form(...),  # contains {{header}}, {{footer}}, {{navigation}}, {{content}}
    session: Session = Depends(get_session),
):
    header = session.get(Component, header_id)
    footer = session.get(Component, footer_id)
    nav = session.get(Component, navigation_id)
    if not header or not footer or not nav:
        raise HTTPException(status_code=400, detail="Invalid component id(s)")

    compiled = render_template(
        raw_html,
        {
            "header": header.compiled_html,
            "footer": footer.compiled_html,
            "navigation": nav.compiled_html,
        },
    )
    # Keep {{content}} for later event rendering.
    shell = Shell(
        name=name,
        header_id=header_id,
        footer_id=footer_id,
        navigation_id=navigation_id,
        raw_html=raw_html,
        compiled_html=compiled,
    )
    session.add(shell)
    session.commit()
    session.refresh(shell)
    return shell

@app.get("/shells", response_model=list[Shell])
async def list_shells(session: Session = Depends(get_session)):
    return session.exec(select(Shell)).all()


# ---- Event Endpoints ------------------------------------------------------
@app.post("/events", response_model=Event)
async def create_event(
    name: str = Form(...),
    shell_id: int = Form(...),
    content_html: str = Form(""),  # inserted into {{content}}
    session: Session = Depends(get_session),
):
    shell = session.get(Shell, shell_id)
    if not shell:
        raise HTTPException(status_code=400, detail="Shell not found")

    final_html = render_template(shell.compiled_html, {"content": content_html})
    event = Event(name=name, shell_id=shell_id, content_html=content_html, final_html=final_html)
    session.add(event)
    session.commit()
    session.refresh(event)
    return event

@app.get("/events", response_model=list[Event])
async def list_events(session: Session = Depends(get_session)):
    return session.exec(select(Event)).all()

@app.get("/events/{event_id}/html")
async def get_event_html(event_id: int, session: Session = Depends(get_session)):
    ev = session.get(Event, event_id)
    if not ev:
        raise HTTPException(status_code=404, detail="Event not found")
    return {"html": ev.final_html}

Run backend

cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000


---

Frontend — React (Vite + TypeScript)

Structure

frontend/
  index.html
  package.json
  tsconfig.json
  vite.config.ts
  src/
    main.tsx
    App.tsx
    api.ts
    types.ts
    components/
      NavBar.tsx
      MediaUploader.tsx
      ComponentForm.tsx
      ShellForm.tsx
      EventForm.tsx
      ListPanel.tsx

frontend/package.json

{
  "name": "shell-system-frontend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.26.2"
  },
  "devDependencies": {
    "@types/react": "^18.3.7",
    "@types/react-dom": "^18.3.0",
    "@types/react-router-dom": "^5.3.3",
    "typescript": "^5.5.4",
    "vite": "^5.4.3"
  }
}

frontend/tsconfig.json

{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "jsx": "react-jsx",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "strict": true
  },
  "include": ["src"]
}

frontend/vite.config.ts

import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: { port: 5173 }
})

frontend/index.html

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Shell Builder</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>

frontend/src/types.ts

export type Media = { id: number; filename: string; url: string };
export type ComponentType = 'header' | 'footer' | 'navigation';
export type ComponentT = {
  id: number; name: string; ctype: ComponentType; raw_html: string; compiled_html: string;
};
export type Shell = {
  id: number; name: string; header_id: number; footer_id: number; navigation_id: number; raw_html: string; compiled_html: string;
};
export type EventT = {
  id: number; name: string; shell_id: number; content_html: string; final_html: string;
};

frontend/src/api.ts

const BASE = import.meta.env.VITE_API_BASE || 'http://localhost:8000';

export async function uploadMedia(file: File) {
  const fd = new FormData();
  fd.append('file', file);
  const res = await fetch(`${BASE}/media`, { method: 'POST', body: fd });
  if (!res.ok) throw new Error('Upload failed');
  return res.json();
}

export const listMedia = () => fetch(`${BASE}/media`).then(r => r.json());

export async function createComponent(payload: { name: string; ctype: string; raw_html: string; media_map: Record<string, number> }) {
  const fd = new FormData();
  fd.append('name', payload.name);
  fd.append('ctype', payload.ctype);
  fd.append('raw_html', payload.raw_html);
  fd.append('media_map', JSON.stringify(payload.media_map || {}));
  const res = await fetch(`${BASE}/components`, { method: 'POST', body: fd });
  if (!res.ok) throw new Error('Component create failed');
  return res.json();
}

export const listComponents = () => fetch(`${BASE}/components`).then(r => r.json());

export async function createShell(payload: { name: string; header_id: number; footer_id: number; navigation_id: number; raw_html: string }) {
  const fd = new FormData();
  fd.append('name', payload.name);
  fd.append('header_id', String(payload.header_id));
  fd.append('footer_id', String(payload.footer_id));
  fd.append('navigation_id', String(payload.navigation_id));
  fd.append('raw_html', payload.raw_html);
  const res = await fetch(`${BASE}/shells`, { method: 'POST', body: fd });
  if (!res.ok) throw new Error('Shell create failed');
  return res.json();
}

export const listShells = () => fetch(`${BASE}/shells`).then(r => r.json());

export async function createEvent(payload: { name: string; shell_id: number; content_html: string }) {
  const fd = new FormData();
  fd.append('name', payload.name);
  fd.append('shell_id', String(payload.shell_id));
  fd.append('content_html', payload.content_html);
  const res = await fetch(`${BASE}/events`, { method: 'POST', body: fd });
  if (!res.ok) throw new Error('Event create failed');
  return res.json();
}

export const listEvents = () => fetch(`${BASE}/events`).then(r => r.json());
export const getEventHTML = (id: number) => fetch(`${BASE}/events/${id}/html`).then(r => r.json());

frontend/src/main.tsx

import React from 'react'
import ReactDOM from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import App from './App'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
)

frontend/src/components/NavBar.tsx

import { Link, useLocation } from 'react-router-dom'

export default function NavBar() {
  const { pathname } = useLocation();
  const tab = (to: string, label: string) => (
    <Link to={to} style={{
      padding: '8px 12px',
      textDecoration: 'none',
      borderRadius: 8,
      background: pathname === to ? '#e3e3ff' : '#f5f5f5',
      border: '1px solid #ddd',
      marginRight: 8,
      color: '#333'
    }}>{label}</Link>
  )
  return (
    <div style={{display:'flex',gap:8, padding:12, borderBottom:'1px solid #eee', position:'sticky', top:0, background:'#fff', zIndex:1}}>
      {tab('/', 'Media')}
      {tab('/components', 'Components')}
      {tab('/shells', 'Shells')}
      {tab('/events', 'Events')}
    </div>
  )
}

frontend/src/components/ListPanel.tsx

import React from 'react'

export default function ListPanel({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <div style={{ marginTop: 24 }}>
      <h3>{title}</h3>
      <div style={{ display: 'grid', gap: 8 }}>{children}</div>
    </div>
  );
}

frontend/src/components/MediaUploader.tsx

import { useEffect, useState } from 'react'
import { uploadMedia, listMedia } from '../api'
import { Media } from '../types'
import ListPanel from './ListPanel'

export default function MediaUploader() {
  const [file, setFile] = useState<File | null>(null)
  const [medias, setMedias] = useState<Media[]>([])

  const refresh = async () => setMedias(await listMedia())
  useEffect(() => { refresh() }, [])

  const onUpload = async () => {
    if (!file) return
    await uploadMedia(file)
    setFile(null)
    await refresh()
  }

  return (
    <div>
      <h2>Media</h2>
      <div style={{ display:'flex', alignItems:'center', gap:8 }}>
        <input type="file" onChange={e => setFile(e.target.files?.[0] || null)} />
        <button onClick={onUpload}>Upload</button>
      </div>
      <ListPanel title="Uploaded">
        {medias.map(m => (
          <div key={m.id} style={{border:'1px solid #ddd', borderRadius:8, padding:8}}>
            <div><b>{m.filename}</b></div>
            <div><small>{m.url}</small></div>
            <div style={{marginTop:6}}><img src={`http://localhost:8000${m.url}`} style={{maxHeight:80}} /></div>
          </div>
        ))}
      </ListPanel>
    </div>
  )
}

frontend/src/components/ComponentForm.tsx

import { useEffect, useMemo, useState } from 'react'
import { createComponent, listComponents, listMedia } from '../api'
import { ComponentT, ComponentType, Media } from '../types'
import ListPanel from './ListPanel'

export default function ComponentForm() {
  const [name, setName] = useState('')
  const [ctype, setCtype] = useState<ComponentType>('header')
  const [rawHtml, setRawHtml] = useState('<header><img src="{{logo}}" alt="logo"/></header>')
  const [media, setMedia] = useState<Media[]>([])
  const [mapRows, setMapRows] = useState<{ var: string; mediaId: number | '' }[]>([{ var: 'logo', mediaId: '' }])
  const [components, setComponents] = useState<ComponentT[]>([])

  const refresh = async () => setComponents(await listComponents())
  useEffect(() => { listMedia().then(setMedia); refresh() }, [])

  const media_map = useMemo(() => Object.fromEntries(mapRows.filter(r => r.var && r.mediaId).map(r => [r.var, Number(r.mediaId)])), [mapRows])

  const submit = async () => {
    if (!name) return alert('Name required')
    const comp = await createComponent({ name, ctype, raw_html: rawHtml, media_map })
    setName(''); setRawHtml(''); setMapRows([{ var: '', mediaId: '' }]); setCtype('header')
    await refresh()
    alert('Component created')
  }

  return (
    <div>
      <h2>Components</h2>
      <div style={{display:'grid', gap:8, maxWidth:800}}>
        <input placeholder="Name" value={name} onChange={e=>setName(e.target.value)} />
        <select value={ctype} onChange={e=>setCtype(e.target.value as ComponentType)}>
          <option value="header">header</option>
          <option value="footer">footer</option>
          <option value="navigation">navigation</option>
        </select>
        <textarea rows={6} placeholder="Raw HTML with {{vars}}" value={rawHtml} onChange={e=>setRawHtml(e.target.value)} />
        <div>
          <b>Media Variable Mapping</b>
          {mapRows.map((row, idx) => (
            <div key={idx} style={{display:'flex', gap:8, marginTop:6}}>
              <input placeholder="var name (e.g., logo)" value={row.var} onChange={e=>{
                const v=[...mapRows]; v[idx].var=e.target.value; setMapRows(v);
              }} />
              <select value={row.mediaId} onChange={e=>{ const v=[...mapRows]; v[idx].mediaId = e.target.value ? Number(e.target.value) : ''; setMapRows(v); }}>
                <option value="">(select media)</option>
                {media.map(m => <option value={m.id} key={m.id}>{m.filename}</option>)}
              </select>
              <button onClick={()=> setMapRows(r=> r.filter((_,i)=>i!==idx))}>Remove</button>
            </div>
          ))}
          <button onClick={()=> setMapRows(r=> [...r, { var:'', mediaId:'' }])} style={{marginTop:6}}>+ Add Mapping</button>
        </div>
        <button onClick={submit}>Save Component</button>
      </div>

      <ListPanel title="Existing Components">
        {components.map(c => (
          <details key={c.id} style={{border:'1px solid #ddd', borderRadius:8, padding:8}}>
            <summary><b>#{c.id}</b> {c.name} ({c.ctype})</summary>
            <div style={{display:'grid', gap:6, marginTop:8}}>
              <div><b>Raw</b><pre style={{whiteSpace:'pre-wrap'}}>{c.raw_html}</pre></div>
              <div><b>Compiled</b><div dangerouslySetInnerHTML={{__html: c.compiled_html}} />
              </div>
            </div>
          </details>
        ))}
      </ListPanel>
    </div>
  )
}

frontend/src/components/ShellForm.tsx

import { useEffect, useState } from 'react'
import { createShell, listComponents, listShells } from '../api'
import { ComponentT, Shell } from '../types'
import ListPanel from './ListPanel'

export default function ShellForm() {
  const [name, setName] = useState('')
  const [components, setComponents] = useState<ComponentT[]>([])
  const [headerId, setHeaderId] = useState<number | ''>('')
  const [footerId, setFooterId] = useState<number | ''>('')
  const [navId, setNavId] = useState<number | ''>('')
  const [rawHtml, setRawHtml] = useState('<div class="page">\n  <div class="nav">{{navigation}}</div>\n  {{header}}\n  <main>{{content}}</main>\n  {{footer}}\n</div>')
  const [shells, setShells] = useState<Shell[]>([])

  const refresh = async () => setShells(await listShells())
  useEffect(() => { listComponents().then(setComponents); refresh() }, [])

  const compOf = (t: 'header' | 'footer' | 'navigation') => components.filter(c => c.ctype === t)

  const submit = async () => {
    if (!name || !headerId || !footerId || !navId) return alert('Fill all fields')
    await createShell({ name, header_id: Number(headerId), footer_id: Number(footerId), navigation_id: Number(navId), raw_html: rawHtml })
    setName(''); setHeaderId(''); setFooterId(''); setNavId(''); setRawHtml('')
    await refresh()
    alert('Shell created')
  }

  return (
    <div>
      <h2>Shells</h2>
      <div style={{display:'grid', gap:8, maxWidth:900}}>
        <input placeholder="Name" value={name} onChange={e=>setName(e.target.value)} />
        <div style={{display:'grid', gridTemplateColumns:'1fr 1fr 1fr', gap:8}}>
          <select value={headerId} onChange={e=>setHeaderId(e.target.value ? Number(e.target.value) : '')}>
            <option value="">(header)</option>
            {compOf('header').map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
          </select>
          <select value={navId} onChange={e=>setNavId(e.target.value ? Number(e.target.value) : '')}>
            <option value="">(navigation)</option>
            {compOf('navigation').map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
          </select>
          <select value={footerId} onChange={e=>setFooterId(e.target.value ? Number(e.target.value) : '')}>
            <option value="">(footer)</option>
            {compOf('footer').map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
          </select>
        </div>
        <textarea rows={8} placeholder="Layout HTML with {{header}} {{navigation}} {{footer}} {{content}}" value={rawHtml} onChange={e=>setRawHtml(e.target.value)} />
        <button onClick={submit}>Save Shell</button>
      </div>

      <ListPanel title="Existing Shells">
        {shells.map(s => (
          <details key={s.id} style={{border:'1px solid #ddd', borderRadius:8, padding:8}}>
            <summary><b>#{s.id}</b> {s.name}</summary>
            <div style={{display:'grid', gap:6, marginTop:8}}>
              <div><b>Raw</b><pre style={{whiteSpace:'pre-wrap'}}>{s.raw_html}</pre></div>
              <div><b>Compiled (keeps {{`{{content}}`}})</b>
                <div dangerouslySetInnerHTML={{__html: s.compiled_html.replace('{{content}}', '<em>(content goes here)</em>')}} />
              </div>
            </div>
          </details>
        ))}
      </ListPanel>
    </div>
  )
}

frontend/src/components/EventForm.tsx

import { useEffect, useState } from 'react'
import { createEvent, getEventHTML, listShells, listEvents } from '../api'
import { Shell, EventT } from '../types'
import ListPanel from './ListPanel'

export default function EventForm() {
  const [name, setName] = useState('')
  const [shells, setShells] = useState<Shell[]>([])
  const [shellId, setShellId] = useState<number | ''>('')
  const [content, setContent] = useState('<section><h1>My Event</h1><p>Hello world</p></section>')
  const [events, setEvents] = useState<EventT[]>([])
  const [previewHTML, setPreviewHTML] = useState('')

  const refresh = async () => setEvents(await listEvents())
  useEffect(() => { listShells().then(setShells); refresh() }, [])

  const submit = async () => {
    if (!name || !shellId) return alert('Fill all fields')
    await createEvent({ name, shell_id: Number(shellId), content_html: content })
    setName(''); setShellId(''); setContent('')
    await refresh()
    alert('Event created')
  }

  const preview = async (id: number) => {
    const { html } = await getEventHTML(id)
    setPreviewHTML(html)
  }

  return (
    <div>
      <h2>Events</h2>
      <div style={{display:'grid', gap:8, maxWidth:900}}>
        <input placeholder="Name" value={name} onChange={e=>setName(e.target.value)} />
        <select value={shellId} onChange={e=>setShellId(e.target.value ? Number(e.target.value) : '')}>
          <option value="">(select shell)</option>
          {shells.map(s => <option key={s.id} value={s.id}>{s.name}</option>)}
        </select>
        <textarea rows={6} placeholder="Event content HTML" value={content} onChange={e=>setContent(e.target.value)} />
        <button onClick={submit}>Save Event</button>
      </div>

      <ListPanel title="Existing Events">
        {events.map(ev => (
          <div key={ev.id} style={{border:'1px solid #ddd', borderRadius:8, padding:8}}>
            <div><b>#{ev.id}</b> {ev.name}</div>
            <div style={{display:'flex', gap:8, marginTop:6}}>
              <button onClick={()=>preview(ev.id)}>Preview</button>
            </div>
          </div>
        ))}
      </ListPanel>

      {previewHTML && (
        <div style={{marginTop:16}}>
          <h3>Preview</h3>
          <div style={{border:'1px solid #ddd', padding:12}} dangerouslySetInnerHTML={{__html: previewHTML}} />
        </div>
      )}
    </div>
  )
}

frontend/src/App.tsx

import { Routes, Route } from 'react-router-dom'
import NavBar from './components/NavBar'
import MediaUploader from './components/MediaUploader'
import ComponentForm from './components/ComponentForm'
import ShellForm from './components/ShellForm'
import EventForm from './components/EventForm'

export default function App() {
  return (
    <div>
      <NavBar />
      <div style={{ padding: 16 }}>
        <Routes>
          <Route path="/" element={<MediaUploader />} />
          <Route path="/components" element={<ComponentForm />} />
          <Route path="/shells" element={<ShellForm />} />
          <Route path="/events" element={<EventForm />} />
        </Routes>
      </div>
    </div>
  )
}
