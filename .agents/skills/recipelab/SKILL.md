---
name: recipelab
description: >
  Use this skill when working on the RecipeLab project. Covers architecture,
  agent patterns, MCP tool definitions, FastAPI endpoints, data models,
  and frontend conventions. Activate for tasks like "add an endpoint",
  "add an MCP tool", "extend the agent", "add a recipe feature", or
  "debug a chat/agent flow".
---

# RecipeLab — Project Skill Reference

RecipeLab is a **multi-agent culinary assistant** built with:
- **Backend**: Python + FastAPI (`backend.py`)
- **AI Orchestration**: Google Antigravity SDK (`google.antigravity`)
- **MCP Servers**: Two FastMCP stdio servers (`recipe_db_mcp.py`, `user_profile_mcp.py`)
- **Frontend**: Vanilla HTML/CSS/JS served as static files from `frontend/`
- **Data**: JSON flat files (`mock_recipes.json`, `mock_user_store.json`)

---

## Project Structure

```
RecipeLab/
├── backend.py            # FastAPI app + Antigravity agent + REST endpoints
├── recipe_db_mcp.py      # MCP server: recipe search + internet fallback tools
├── user_profile_mcp.py   # MCP server: pantry, favorites, shopping list tools
├── mock_recipes.json     # Local recipe database (array of recipe objects)
├── mock_user_store.json  # Persistent user state (pantry, restrictions, etc.)
├── start.py              # Convenience launcher
├── .env                  # Environment variables
├── frontend/
│   ├── index.html        # Single-page app shell
│   ├── app.js            # All frontend logic
│   └── style.css         # Premium dark glassmorphism design
└── .agents/
    └── skills/recipelab/SKILL.md
```

---

## Agent Architecture

### Concierge Agent (Orchestrator)
Defined in `backend.py` via `LocalAgentConfig`. Uses `gemini-2.0-flash`.

- Always fetches user profile first using `get_user_profile`
- Uses explicitly mentioned ingredients for search; falls back to pantry
- Spawns sub-agents via `start_subagent`
- Enforces dietary restrictions strictly

### Sub-Agents (spawned on demand)

| Sub-Agent | Responsibility | MCP Tool |
|---|---|---|
| Recipe Finder Agent | Search + diet filter | `search_recipes_by_ingredients`, `filter_by_diet` |
| Preferences Agent | Star/unstar recipes | `toggle_favorite_recipe` |
| List Manager Agent | Shopping list updates | `add_to_shopping_list` |

### MCP Tools

#### recipe_db (recipe_db_mcp.py)
- `search_recipes_by_ingredients(ingredients, strict_mode=False)` — local DB + internet fallback if best match < 95%
- `filter_by_diet(recipes, restrictions)` — dietary tag filtering

#### user_profile (user_profile_mcp.py)
- `get_user_profile()` — returns pantry, restrictions, shopping_list, favorites
- `add_to_shopping_list(ingredients)` — adds missing items
- `toggle_favorite_recipe(recipe_id, is_starred)` — favorites toggle
- `update_pantry(ingredients)` — replaces pantry
- `update_restrictions(restrictions)` — replaces dietary restrictions

---

## Data Models

### Recipe Object
```json
{
  "id": "recipe_1",
  "name": "Spinach Tomato Omelette",
  "ingredients": ["eggs", "spinach", "tomatoes"],
  "instructions": "1. Beat eggs...",
  "is_gluten_free": true,
  "is_dairy_free": true,
  "is_vegetarian": true,
  "is_vegan": false
}
```

### User Store
```json
{
  "pantry": ["eggs", "spinach"],
  "restrictions": ["Vegetarian"],
  "shopping_list": ["butter"],
  "favorites": ["recipe_1"]
}
```

### Recipe ID Conventions
- Local: `"recipe_1"`, `"recipe_2"`, ...
- Spoonacular: `"internet_spoonacular_<id>"`
- TheMealDB: `"internet_mealdb_<id>"`

---

## REST API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | /api/profile | Get user profile |
| GET | /api/recipes | Get all local recipes |
| GET | /api/recipes/search | Search (?ingredients=&strict=&diet=) |
| GET | /api/recipes/{recipe_id} | Get single recipe |
| GET | /api/settings | Get Spoonacular key |
| POST | /api/settings | Save Spoonacular key |
| POST | /api/pantry | Replace pantry |
| POST | /api/pantry/add | Add single pantry item |
| POST | /api/restrictions | Replace restrictions |
| POST | /api/shopping_list/add | Add to shopping list |
| POST | /api/shopping_list/set | Replace shopping list |
| POST | /api/shopping_list/clear | Clear shopping list |
| DELETE | /api/shopping_list/{item} | Remove one item |
| POST | /api/favorites/toggle | Star/unstar recipe |
| POST | /api/chat | Chat → SSE stream |

### Chat SSE Events
```
data: {"type": "thought", "content": "..."}   // reasoning
data: {"type": "text", "content": "..."}       // response text
data: {"type": "done"}                          // end of stream
```

---

## Environment Variables
```
GEMINI_API_KEY=...         # Required for AI chat
SPOONACULAR_API_KEY=...    # Optional: premium internet recipe search
```

---

## Common Development Patterns

### Add an MCP Tool
1. Add `@mcp.tool()` function with `Args:` docstring to relevant MCP file
2. Update Concierge `system_instructions` in `backend.py`

### Add a REST Endpoint
1. Add Pydantic `BaseModel` for request body
2. Add `@app.get/post/delete(...)` async function
3. Use `get_stored_profile()` / `save_stored_profile()` for user data

### Add a Recipe Field
1. Add to `mock_recipes.json`
2. Propagate in `recipe_db_mcp.py` result dicts
3. Update internet recipe adapters in `backend.py`
4. Update `app.js` if it should be displayed

### Dietary Tag Inference
Sets in `recipe_db_mcp.py`:
- `MEATS_AND_FISH` → non-vegetarian, non-vegan
- `ANIMAL_PRODUCTS` → non-vegan
- `DAIRY_ITEMS` → non-dairy-free
- `GLUTEN_ITEMS` → non-gluten-free

### Run Locally
```bash
python start.py
# App at http://127.0.0.1:8000
```

---

## Key Design Decisions
- **90s timeout + fallback**: `execute_mock_fallback()` ensures the app works without a live API key
- **MCP over direct imports**: Allows Antigravity SDK to spawn true subagents with isolated tool access
- **Flat JSON files**: No database dependency for local development
- **Static file serving**: FastAPI mounts `frontend/` at `/` — no separate dev server needed
