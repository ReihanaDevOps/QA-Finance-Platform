docker context inspect desktop-linux --format "{{.Endpoints.docker.Host}}"; [System.IO.Directory]::GetFiles("\\.\pipe\") | Select-String -Pattern "docker"
Your QA Platform
       │
       │ URL + username + password
       ▼
Local QA Agent
       │
       ├── Playwright → opens browser
       │
       ├── Claude Code → AI reasoning
       │
       ├── DOM collector
       ├── Screenshot collector
       └── Network collector
       │
       ▼
Application Map
       │
       ▼
Claude analyzes
       │
       ├── Functional
       ├── Performance
       └── Security
       │
       ▼
JSON
       │
       ▼
Your QA Backend
       │
       ▼
Client UI
