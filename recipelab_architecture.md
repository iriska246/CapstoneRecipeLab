# RecipeLab — Architecture Diagram

## System Overview

```mermaid
graph TB
    subgraph Browser["🌐 Browser (frontend/)"]
        UI["index.html\nSingle-Page App"]
        JS["app.js\nChat · Pantry · Shopping · Favorites"]
        CSS["style.css\nDark Glassmorphism UI"]
    end

    subgraph FastAPI["⚙️ FastAPI Backend (backend.py)"]
        REST["REST Endpoints\n/api/profile · /api/recipes\n/api/pantry · /api/shopping_list\n/api/favorites · /api/settings"]
        CHAT["/api/chat\nSSE Streaming Endpoint"]
        FALLBACK["execute_mock_fallback()\nLocal Pattern-Matching Engine\n(90s timeout fallback)"]
    end

    subgraph AgentLayer["🤖 Antigravity Agent Layer"]
        CONCIERGE["Concierge Agent\ngemini-2.0-flash\nOrchestrator"]
        RF["Recipe Finder\nSub-Agent"]
        PA["Preferences\nSub-Agent"]
        LM["List Manager\nSub-Agent"]
    end

    subgraph MCPServers["🔌 MCP Servers (stdio)"]
        RecipeMCP["recipe_db_mcp.py\n─────────────────\nsearch_recipes_by_ingredients\nfilter_by_diet"]
        UserMCP["user_profile_mcp.py\n─────────────────\nget_user_profile\nadd_to_shopping_list\ntoggle_favorite_recipe\nupdate_pantry\nupdate_restrictions"]
    end

    subgraph DataLayer["💾 Data Layer (JSON)"]
        Recipes["mock_recipes.json\nLocal Recipe Database"]
        UserStore["mock_user_store.json\nUser State\n(pantry · restrictions\nshopping_list · favorites)"]
        EnvFile[".env\nGEMINI_API_KEY\nSPOONACULAR_API_KEY"]
    end

    subgraph Internet["🌍 External APIs"]
        Spoon["Spoonacular API\n(optional, premium)"]
        MealDB["TheMealDB API\n(free fallback)"]
    end

    %% Browser ↔ Backend
    UI --> JS
    JS <-->|"fetch() REST calls"| REST
    JS <-->|"SSE stream"| CHAT

    %% Backend data access
    REST <-->|read/write| UserStore
    REST <-->|read| Recipes

    %% Chat flow
    CHAT -->|"tries real agent\n(90s timeout)"| CONCIERGE
    CHAT -->|"on timeout/quota"| FALLBACK
    FALLBACK <-->|read/write| UserStore
    FALLBACK <-->|read| Recipes

    %% Agent orchestration
    CONCIERGE -->|"spawns"| RF
    CONCIERGE -->|"spawns"| PA
    CONCIERGE -->|"spawns"| LM

    %% Agent → MCP
    RF <-->|MCP stdio| RecipeMCP
    PA <-->|MCP stdio| UserMCP
    LM <-->|MCP stdio| UserMCP
    CONCIERGE <-->|MCP stdio| UserMCP

    %% MCP → Data
    RecipeMCP <-->|read| Recipes
    RecipeMCP -->|"if best match < 95%"| Spoon
    RecipeMCP -->|"if no Spoon key"| MealDB
    UserMCP <-->|read/write| UserStore

    %% Styling
    classDef browser fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    classDef backend fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    classDef agent fill:#3b1f5e,stroke:#a855f7,color:#e2e8f0
    classDef mcp fill:#3a2a1a,stroke:#f97316,color:#e2e8f0
    classDef data fill:#1a2a3a,stroke:#64748b,color:#e2e8f0
    classDef internet fill:#2a1a1a,stroke:#ef4444,color:#e2e8f0

    class UI,JS,CSS browser
    class REST,CHAT,FALLBACK backend
    class CONCIERGE,RF,PA,LM agent
    class RecipeMCP,UserMCP mcp
    class Recipes,UserStore,EnvFile data
    class Spoon,MealDB internet
```

---

## Request Lifecycle: Chat Message

```mermaid
sequenceDiagram
    participant U as 👤 User (Browser)
    participant B as ⚙️ FastAPI /api/chat
    participant A as 🤖 Concierge Agent
    participant MCP as 🔌 MCP Servers
    participant SA as 🤖 Sub-Agent
    participant FB as 🔄 Fallback Engine

    U->>B: POST /api/chat {message, api_key}
    B->>A: agent.chat(message) [90s timeout]

    A->>MCP: get_user_profile()
    MCP-->>A: {pantry, restrictions, favorites, shopping_list}

    alt Recipe search intent
        A->>SA: spawn "Recipe Finder Agent"
        SA->>MCP: search_recipes_by_ingredients(ingredients)
        MCP-->>SA: results (local + internet if needed)
        SA->>MCP: filter_by_diet(results, restrictions)
        MCP-->>SA: filtered results
        SA-->>A: recipe list
    else Favorites intent
        A->>SA: spawn "Preferences Agent"
        SA->>MCP: toggle_favorite_recipe(id, starred)
        MCP-->>SA: success msg
        SA-->>A: confirmation
    else Shopping list intent
        A->>SA: spawn "List Manager Agent"
        SA->>MCP: add_to_shopping_list(items)
        MCP-->>SA: success msg
        SA-->>A: confirmation
    end

    A-->>B: streamed thoughts + text chunks
    B-->>U: SSE events {type: thought/text/done}

    alt Agent timeout or quota error
        B->>FB: execute_mock_fallback(message)
        FB-->>B: pattern-matched response
        B-->>U: SSE (simulated typing stream)
    end
```

---

## Data Flow: Recipe Search

```mermaid
flowchart LR
    Query["User query\n'eggs, spinach, tomatoes'"]
    Extract["Extract ingredients\nfrom message text"]
    LocalDB["Search mock_recipes.json\nMatch % per recipe"]
    Decision{Best match\n≥ 95%?}
    Return["Return local results"]
    Internet["Fetch from internet"]
    Spoon{Spoonacular\nkey set?}
    SpoonAPI["Spoonacular API\ncomplexSearch"]
    MealDB["TheMealDB API\nfilter + lookup"]
    DietFilter["filter_by_diet()\napply restrictions"]
    Final["Ranked results\n(match % desc)"]

    Query --> Extract --> LocalDB --> Decision
    Decision -- Yes --> Return --> DietFilter
    Decision -- No --> Internet --> Spoon
    Spoon -- Yes --> SpoonAPI --> DietFilter
    Spoon -- No --> MealDB --> DietFilter
    DietFilter --> Final
```

---

## Component Summary

| Layer | File(s) | Technology |
|---|---|---|
| **Frontend** | `frontend/index.html`, `app.js`, `style.css` | Vanilla HTML/CSS/JS, Outfit font |
| **Backend API** | `backend.py` | Python, FastAPI, Uvicorn |
| **AI Orchestration** | `backend.py` | Google Antigravity SDK, `gemini-2.0-flash` |
| **MCP: Recipes** | `recipe_db_mcp.py` | FastMCP, httpx |
| **MCP: User Profile** | `user_profile_mcp.py` | FastMCP |
| **Local Data** | `mock_recipes.json`, `mock_user_store.json` | JSON flat files |
| **Config** | `.env` | python-dotenv |
| **Launcher** | `start.py` | subprocess / uvicorn |
