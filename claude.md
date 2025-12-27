# Instructions for Claude

## Project Context
This is a **Software Engineer Study** repository with multiple roadmaps and language support.

### Structure
```
software-engineer-study/
├── roadmaps/
│   ├── backend/{ru,en}/      # Backend Developer
│   ├── frontend/{ru,en}/     # Frontend Developer
│   ├── devops/{ru,en}/       # DevOps Engineer
│   ├── blockchain/{ru,en}/   # Blockchain Developer
│   ├── gamedev/{ru,en}/      # Game Developer
└── └── ux-ui/{ru,en}/        # UX/UI Designer
```

### Two-file System
Each language folder uses:
| File | Purpose | `[x]` means |
|------|---------|-------------|
| `roadmap.md` | Personal learning progress | User studied and understood the topic |
| `readme.md` | Content availability | File has >10 lines of content |

### Current Session State
- **Current Roadmap**: backend
- **Current Language**: ru
- **Base Path**: `roadmaps/backend/ru/`

## User Preferences

### Language
- **User knows Russian better than English**
- **Study and explain topics in Russian**
- Code examples and technical terms can stay in English

### Study Approach
1. **Step-by-step learning** - go through topics one by one following the roadmap order
2. **Write notes as we go** - fill in the `.md` files with study material during sessions
3. **Practice when needed** - some topics require hands-on exercises, ask user if they want practice tasks
4. **Track progress separately**:
   - Mark `[x]` in `readme.md` when content is written to a file (>10 lines)
   - Mark `[x]` in `roadmap.md` when user confirms they have studied the topic

### Session Continuity
Before starting a new study session:
1. Check current roadmap's `roadmap.md` to see learning progress
2. Find the next uncompleted topic `[ ]`
3. Ask user to confirm we continue from there or if they want to review/practice something

## How to Study a Topic

1. **Check content availability** in `readme.md` — if topic has `[ ]`, fill it first
2. **Read the current topic file** to see existing notes
3. **Explain the concept in the current language** with examples
4. **Write summary notes** to the topic's `.md` file (if not already filled)
5. **Mark `[x]` in `readme.md`** after writing content (if file now has >10 lines)
6. **Offer practice** if the topic benefits from hands-on exercises
7. **Mark `[x]` in `roadmap.md`** when user confirms they understood the topic

## Content Status Rules
- A file is considered "filled" if it has **more than 10 lines** of content
- When filling a topic file, update the corresponding `readme.md` to mark it `[x]`
- The `readme.md` reflects the state of the repository, not user's personal progress

## Content Generation

### Generating Content
When filling topic files, create **detailed material** including:
- Deep explanation of the concept (in current language)
- Practical code examples
- Best practices and common mistakes
- Links to additional resources (optional)

### Auto-fill Rule
**IMPORTANT**: If user wants to study a topic that has NO content (check `readme.md`):
1. First, fill the topic with content
2. Then proceed with studying
3. Never study from empty files

## Parallel Content Generation

When user requests parallel filling:
1. Launch multiple Task agents (one per folder)
2. Each agent fills all topics in its assigned folder
3. Update `readme.md` in each folder after completion
4. Report summary when all agents finish

## Current Progress
- Check `{base_path}/roadmap.md` for **learning progress** (`[x]` = studied)
- Check `{base_path}/readme.md` for **content status** (`[x]` = has content)

## Quick Commands

### Navigation Commands
- "roadmap <name>" / "роадмап <name>" - switch to roadmap (backend/frontend/devops/blockchain/gamedev/ux-ui)
  - Example: "roadmap frontend" or "роадмап devops"
- "lang <code>" / "язык <code>" - switch language (ru/en)
  - Example: "lang en" or "язык ru"

### Study Commands
- "продолжим" / "continue" - resume from last topic in current roadmap
- "практика" / "practice" - give exercises for current topic
- "повтори" / "review" - review previously learned topic
- "статус" / "status" - show current progress

### Content Commands
- "заполни <путь>" / "fill <path>" - fill specific topic with content
  - Example: "заполни 03-git" or "fill 03-version-control-systems"
- "заполни папку <путь>" / "fill folder <path>" - fill all topics in a folder
  - Example: "заполни папку 04-postgresql" or "fill folder 08-apis"
- "заполни параллельно <п1>, <п2>, ..." / "fill parallel <p1>, <p2>, ..." - fill multiple folders in parallel
  - Example: "заполни параллельно 05, 06, 07" or "fill parallel 09, 10, 11"
