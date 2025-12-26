# Instructions for Claude

## Project Context
This is a **Backend Developer study roadmap** based on roadmap.sh. The structure includes:
- `roadmap.md` - tracks **user's personal learning progress** (what they have studied)
- `readme.md` - tracks **content status** (which files are filled with basic information)
- 25 topic folders (01-internet through 25-cyber-security) + extras
- Each folder has its own `roadmap.md` and `readme.md` linking to subtopic files
- `.md` files contain study notes

### Two-file System
| File | Purpose | `[x]` means |
|------|---------|-------------|
| `roadmap.md` | Personal learning progress | User studied and understood the topic |
| `readme.md` | Content availability | File has >10 lines of content |

## User Preferences

### Language
- **User knows Russian better than English**
- **Study and explain topics in Russian**
- Code examples and technical terms can stay in English

### Study Approach
1. **Step-by-step learning** - we go through topics one by one following the roadmap order
2. **Write notes as we go** - fill in the `.md` files with study material during sessions
3. **Practice when needed** - some topics require hands-on exercises, ask user if they want practice tasks
4. **Track progress separately**:
   - Mark `[x]` in `readme.md` when content is written to a file (>10 lines)
   - Mark `[x]` in `roadmap.md` when user confirms they have studied the topic

### Session Continuity
Before starting a new study session:
1. Check `roadmap.md` to see current learning progress (look for `[x]` marks)
2. Find the next uncompleted topic `[ ]`
3. Ask user to confirm we continue from there or if they want to review/practice something

## How to Study a Topic

1. **Check content availability** in `readme.md` — if topic has `[ ]`, fill it first
2. **Read the current topic file** to see existing notes
3. **Explain the concept in Russian** with examples
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
- Глубокое объяснение концепции
- Практические примеры кода
- Best practices и типичные ошибки
- Ссылки на дополнительные ресурсы (опционально)

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

**Agent prompt template:**
- Fill all `.md` files in the folder with detailed content in Russian
- Check each file's current state before writing
- Mark completed files in `readme.md` with `[x]`

## Current Progress
- Check `roadmap.md` for **learning progress** (`[x]` = studied)
- Check `readme.md` for **content status** (`[x]` = has content)

## Quick Commands

### Study Commands
- "продолжим" / "continue" - resume from last topic
- "практика" / "practice" - give exercises for current topic
- "повтори" / "review" - review previously learned topic
- "статус" / "status" - show current progress

### Content Commands
- "заполни <путь>" / "fill <path>" - fill specific topic with content
  - Example: "заполни 03-git" or "fill 03-version-control-systems"
- "заполни папку <путь>" / "fill folder <path>" - fill all topics in a folder
  - Example: "заполни папку 04-postgresql" or "fill folder 08-apis"
- "заполни параллельно <п1>, <п2>, ..." / "fill parallel <p1>, <p2>, ..." - fill multiple folders in parallel using agents
  - Example: "заполни параллельно 05, 06, 07" or "fill parallel 09, 10, 11"
