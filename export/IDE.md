# IDE Best Practices

Most engineers use about 10% of their IDE. They click through menus, scroll to find things, and manually type what the IDE could generate. This isn't a moral failing — nobody sits you down and says "here are the 20 shortcuts that will change your life." But the gap between someone who knows their IDE and someone who doesn't is enormous. It's not about speed — it's about staying in flow. Every time you reach for the mouse, open a file browser, or manually type an import, you're breaking the thread of thought you were holding.

This guide covers the features, shortcuts, and mental models that separate productive IDE usage from "fancy text editor" usage.

---

## The Core Principle

**Your IDE knows more about your code than you do.** It has indexed every symbol, every reference, every type relationship. The single biggest mistake engineers make is treating their IDE like a text editor with syntax highlighting. Your IDE is an analysis engine. Learn to query it.

---

## The 20 Shortcuts That Matter Most

You don't need to memorize 200 shortcuts. You need 20, and you need them in muscle memory. Everything else is gravy.

### Tier 1: Learn These First (Day 1)

These are non-negotiable. If you don't know these, you're doing manual labor the machine should be doing.

| Action | IntelliJ (macOS) | VS Code (macOS) | Why It Matters |
|---|---|---|---|
| **Go to File** | `Cmd+Shift+O` | `Cmd+P` | Never use the file tree to open files. Type 2-3 letters of the name. |
| **Go to Symbol** | `Cmd+Option+O` | `Cmd+T` | Find any class, method, or variable by name across the entire project. |
| **Go to Definition** | `Cmd+B` | `F12` | Jump to where something is defined. Stop scrolling. |
| **Find Usages** | `Option+F7` | `Shift+F12` | "Who calls this method?" is the most important question in any codebase. |
| **Search Everywhere** | `Shift+Shift` | `Cmd+Shift+P` | The universal entry point. If you forget everything else, remember this one. |
| **Back / Forward** | `Cmd+[` / `Cmd+]` | `Ctrl+-` / `Ctrl+Shift+-` | Navigate your jump history like browser tabs. |
| **Quick Fix / Action** | `Option+Enter` | `Cmd+.` | The IDE's suggestion engine. Fix errors, add imports, generate code — all from one shortcut. |

### Tier 2: Learn These Next (Week 1)

| Action | IntelliJ (macOS) | VS Code (macOS) | Why It Matters |
|---|---|---|---|
| **Rename** | `Shift+F6` | `F2` | Renames everywhere — references, imports, file names. Never find-and-replace a symbol name. |
| **Extract Variable** | `Cmd+Option+V` | `Cmd+Shift+R` → extract | Pull an expression into a named variable. Makes code readable. |
| **Extract Method** | `Cmd+Option+M` | `Cmd+Shift+R` → extract | Pull a block into a method. The #1 refactoring you'll ever use. |
| **Inline** | `Cmd+Option+N` | — | The inverse of extract. Remove unnecessary variables/methods. |
| **Move Line Up/Down** | `Option+Shift+↑/↓` | `Option+↑/↓` | Reorder code without cut/paste. |
| **Duplicate Line** | `Cmd+D` | `Option+Shift+↓` | Faster than copy/paste for nearby duplication. |
| **Delete Line** | `Cmd+Backspace` | `Cmd+Shift+K` | One keystroke, not select-all-then-delete. |
| **Multiple Cursors** | `Option+Click` | `Option+Click` | Edit multiple lines simultaneously. Game-changer for repetitive edits. |
| **Select Next Occurrence** | `Ctrl+G` | `Cmd+D` | Select matching text incrementally. Faster than find-and-replace for small batches. |
| **Toggle Comment** | `Cmd+/` | `Cmd+/` | Comment/uncomment lines or blocks. |

### Tier 3: Power User (Month 1)

| Action | IntelliJ (macOS) | VS Code (macOS) | Why It Matters |
|---|---|---|---|
| **Show Type Hierarchy** | `Ctrl+H` | — | Visualize inheritance. Essential for understanding frameworks. |
| **Show Call Hierarchy** | `Ctrl+Option+H` | `Shift+Option+H` | "What calls this, and what does this call?" Trace execution paths without running code. |
| **Find in Path (Project-wide search)** | `Cmd+Shift+F` | `Cmd+Shift+F` | Grep across the project with regex, scope filters, and file masks. |
| **Run Current Test** | `Ctrl+Shift+R` | `Ctrl+Shift+T` (with extension) | Run the test under your cursor. Not the whole suite. Tight feedback loop. |
| **Evaluate Expression (Debug)** | `Option+F8` | Debug Console | Run arbitrary expressions against the current debug context. |
| **Structural Search** | `Cmd+Shift+S` (IntelliJ) | — | Search by code pattern, not text. "Find all `if` blocks that return null." |

---

## Navigation: The Most Underused Superpower

Navigation is the single highest-ROI skill to develop. Most debugging and code comprehension is navigation — jumping between definitions, usages, implementations, and callers.

### The Navigation Mental Model

Think of navigation as asking your IDE questions:

| Question You Have | IDE Action | How to Invoke |
|---|---|---|
| "What is this thing?" | **Go to Definition** | `Cmd+B` / `F12` |
| "Who uses this?" | **Find Usages** | `Option+F7` / `Shift+F12` |
| "What implements this interface?" | **Go to Implementation** | `Cmd+Option+B` / `Cmd+F12` |
| "What does this class extend?" | **Type Hierarchy** | `Ctrl+H` |
| "Where was I just looking?" | **Navigate Back** | `Cmd+[` / `Ctrl+-` |
| "What's the structure of this file?" | **File Structure** | `Cmd+F12` / `Cmd+Shift+O` |
| "Where is this string used in config?" | **Find in Path** | `Cmd+Shift+F` |
| "What changed recently in this file?" | **Local History / Git Blame** | Right-click → Local History |

### Common Navigation Anti-Patterns

| Anti-Pattern | Why It's Bad | What to Do Instead |
|---|---|---|
| Using the file tree to find files | Slow, breaks focus, doesn't scale | `Cmd+Shift+O` / `Cmd+P` — type partial name |
| Scrolling to find a method | O(n) where n = file length | `Cmd+F12` for file structure, or `Cmd+Shift+O` / `Cmd+T` for symbol search |
| Grep in terminal for "who calls this" | Finds string matches, not semantic usages | Find Usages — it understands scope, overrides, and type relationships |
| Opening the same 3 files repeatedly | Context switching overhead | Pin frequently-used tabs; use recent files (`Cmd+E` / `Ctrl+R`) |
| Manually tracing call chains | Error-prone and exhausting | Call Hierarchy shows the full chain, both directions |

### Bookmarks and TODO Tracking

When investigating a bug or understanding a flow, you'll touch 10+ files. Don't rely on memory.

- **Bookmarks** (`F11` in IntelliJ, `Ctrl+Shift+[number]` with Bookmarks extension in VS Code): Mark important locations and jump back to them.
- **TODO comments**: Most IDEs have a TODO tool window that collects all `// TODO`, `// FIXME`, `// HACK` comments. Use it during code review and investigation.
- **Scratch files** (`Cmd+Shift+N` in IntelliJ): Create disposable files for notes, snippets, or experiments without cluttering your project.

---

## Debugging: Stop Using Print Statements

Print-statement debugging is the #1 productivity killer in software engineering. It requires a modify → compile → run → read cycle for every hypothesis. Your IDE debugger lets you test hypotheses in real time, without modifying code.

### Breakpoint Types You Should Know

| Type | What It Does | When to Use |
|---|---|---|
| **Line Breakpoint** | Pauses execution at a specific line | Basic debugging — inspect state at a known point |
| **Conditional Breakpoint** | Only pauses when a condition is true | "Stop here only when `userId == 42`" — essential for loops and high-traffic code |
| **Log Breakpoint (Tracepoint)** | Logs a message without stopping | Print-statement debugging without modifying code. Remove when done with zero risk. |
| **Exception Breakpoint** | Pauses when a specific exception is thrown | "Where is this NullPointerException actually thrown?" — catches it at the source, not the handler |
| **Method Breakpoint** | Pauses when a method is entered or exited | Useful for interface methods where you don't know which implementation runs |

### The Debugging Workflow

Most engineers set a breakpoint and then stare at variables. That's step 1 of 5.

1. **Reproduce** — Get a reliable reproduction first. If you can't reproduce it, you can't debug it.
2. **Hypothesize** — Form a theory about what's wrong before setting breakpoints. "I think the filter is applied before the sort, so the sort operates on an empty list."
3. **Set Strategic Breakpoints** — Don't scatter breakpoints everywhere. Place them at decision points that confirm or deny your hypothesis.
4. **Inspect State** — Use the Variables panel, Watches, and Evaluate Expression to examine state. Don't just glance — verify your assumptions.
5. **Step Through Selectively** — Use Step Over (`F8`/`F10`) for lines you trust. Use Step Into (`F7`/`F11`) only for the call you're investigating. Use Step Out (`Shift+F8`/`Shift+F11`) when you've seen enough inside a method.

### Debugger Features Most People Miss

| Feature | What It Does | Why You Need It |
|---|---|---|
| **Evaluate Expression** | Run arbitrary code in the current debug context | Test fixes before writing them. `"What if I add 1 to this offset?"` |
| **Watches** | Persistent expressions that update at every breakpoint | Track derived values like `list.size()` or `user.getRole()` across steps |
| **Drop Frame / Reset Frame** | Re-run the current method from the start | Made a mistake stepping? Don't restart — just rewind. |
| **Hot Reload / HotSwap** | Apply code changes without restarting the app | Change a method body and continue debugging. Not all languages support this. |
| **Memory View** (IntelliJ) | See object counts by class | "How many `Order` objects are alive right now?" Catches leaks early. |
| **Stream Debugger** (IntelliJ) | Visualize Java Stream pipeline stages | See what each `.filter()`, `.map()`, `.collect()` step produces |

### The Litmus Test for Print vs. Debugger

> If you're adding `System.out.println` or `console.log` to understand control flow, you should be using the debugger. If you're adding them to log production behavior, you should be using structured logging (see logging.md).

---

## Refactoring: Let the IDE Do the Heavy Lifting

Manual refactoring (find-and-replace, copy-paste-modify) is error-prone. IDE refactorings are semantic — they understand scope, type relationships, and references. They're also undoable.

### The Refactorings You Need

| Refactoring | What It Does | When to Use | Shortcut (IntelliJ) |
|---|---|---|---|
| **Rename** | Renames a symbol and all its references | Always. Never manually rename anything. | `Shift+F6` |
| **Extract Variable** | Pulls an expression into a local variable | When an expression is used more than once, or is hard to read | `Cmd+Option+V` |
| **Extract Method** | Pulls a block of code into a new method | When a method is too long, or a block has a clear single responsibility | `Cmd+Option+M` |
| **Extract Constant** | Pulls a literal into a named constant | Magic numbers, repeated strings | `Cmd+Option+C` |
| **Inline** | Replaces a variable/method with its value/body | When a name adds no clarity, or after an extract went too far | `Cmd+Option+N` |
| **Change Signature** | Modifies method parameters and updates all callers | Adding/removing/reordering parameters safely | `Cmd+F6` |
| **Move** | Moves a class/method to a different package/file | Reorganizing code structure | `F6` |
| **Safe Delete** | Deletes a symbol only if it has no usages | Cleaning up dead code without breaking things | `Cmd+Delete` |

### The Refactoring Anti-Pattern

> **Never use find-and-replace to rename a symbol.** It doesn't understand scope. Renaming a local variable `count` will also rename the `count` in an unrelated method, the `count` column name in a SQL string, and the word "count" in a comment. IDE rename is scope-aware and type-aware.

---

## Git Integration: Your IDE Knows Git

Most engineers alt-tab to a terminal for git operations. Your IDE's git integration is often superior for day-to-day work.

### What the IDE Does Better Than the Terminal

| Operation | Terminal | IDE |
|---|---|---|
| **Blame** | `git blame file.java` — wall of text | Inline annotations next to each line. Hover for commit message, author, date. Click to see the full diff. |
| **Diff** | `git diff` — raw text diff | Side-by-side visual diff with syntax highlighting. Navigate between changes with shortcuts. |
| **Partial Staging (Hunks)** | `git add -p` — interactive but clunky | Click individual lines or hunks to stage. Visual, fast, precise. |
| **Merge Conflicts** | Manual editing with `<<<<<<<` markers | Three-pane merge tool: left (yours), right (theirs), center (result). Accept changes with one click. |
| **Local History** | Nothing — git only tracks commits | IDE tracks every save. Recover code you never committed. This has saved careers. |
| **Interactive Rebase** | `git rebase -i` — vim editing | Visual drag-and-drop of commits. Squash, reorder, edit messages. |
| **Cherry-pick hunks** | `git cherry-pick` — whole commits only | Copy individual changes from one branch to another |

### What the Terminal Does Better

- **Complex branch operations**: rebasing onto specific commits, octopus merges, bisect
- **Scripting and automation**: hooks, aliases, CI/CD pipelines
- **Bulk operations**: operating on many repos, batch renaming branches

**The rule**: Use the IDE for visual, interactive git work (diffs, blame, staging, conflicts). Use the terminal for scripting and complex branch surgery.

### The Killer Feature: Annotate (Blame)

`git blame` in the terminal is ugly and hard to read. IDE blame (called "Annotate" in IntelliJ, "GitLens" in VS Code) is transformative:

- **Hover over an annotation** to see the full commit message, author, and date
- **Click an annotation** to see the full diff for that commit
- **"Show Diff with Previous"** to see what changed in that specific line over time
- **"Annotate Previous Revision"** to peel back layers — "who wrote the line that was here before this change?"

This is how you answer "why does this code exist?" without asking anyone.

---

## What Information Should You Be Getting From Your IDE?

Your IDE is constantly analyzing your code. Most of this analysis is available if you know where to look.

### Passive Information (Always Visible)

| Information | Where to Find It | Why It Matters |
|---|---|---|
| **Errors and warnings** | Editor gutter (red/yellow markers), Problems panel | Don't wait for compile. The IDE catches errors in real time. |
| **Unused code** | Grayed-out symbols | Dead code is a maintenance burden. Delete it. |
| **Type information** | Hover over any expression | "What type does this method return?" — don't guess. |
| **Parameter hints** | Inline hints next to method arguments | `process(true, false, 3)` becomes `process(force: true, retry: false, maxAttempts: 3)` |
| **Inferred types** | Inline hints for `var`/`auto`/`let` declarations | See what the compiler actually inferred without reading the source of the method |
| **Code coverage** | Gutter highlighting (green/red) after running tests with coverage | "Is this branch tested?" — visible at a glance |

### Active Information (You Have to Ask)

| Information | How to Get It | Why It Matters |
|---|---|---|
| **Who calls this method?** | Find Usages (`Option+F7` / `Shift+F12`) | Understand impact before changing anything |
| **What implements this interface?** | Go to Implementation (`Cmd+Option+B`) | "Which concrete class actually runs at runtime?" |
| **What's the class hierarchy?** | Type Hierarchy (`Ctrl+H`) | Understand inheritance chains, especially in framework-heavy code |
| **What does this dependency provide?** | Navigate into library source (decompiled if needed) | Don't guess what a library method does — read its code |
| **What's the data flow?** | Analyze Data Flow (`Ctrl+Shift+F7` in IntelliJ) | "Where does this variable get its value?" — traces assignments backward |
| **What are the recent changes?** | Local History / Git Log for file | "What changed in this file since yesterday?" |
| **What's the test coverage?** | Run with Coverage | "Is this code path tested at all?" |

---

## IDE Configuration: Spend 30 Minutes, Save 30 Hours

### Settings That Matter

| Setting | What to Do | Why |
|---|---|---|
| **Auto-import** | Enable it | Don't manually write import statements. Ever. |
| **Auto-save** | Enable on focus loss | The IDE saves when you switch windows. You lose nothing. |
| **Show whitespace** | Enable for leading spaces/tabs | Catch mixed indentation before it hits code review |
| **Format on save** | Enable with project formatter | Consistent formatting without thinking about it |
| **Optimize imports on save** | Enable | Remove unused imports automatically |
| **Font & line height** | Use a programming font (JetBrains Mono, Fira Code) with ligatures | Ligatures make `=>`, `!=`, `>=` more readable. Higher line height reduces eye strain. |
| **Editor tabs** | Limit to 10-15. Consider "close others" shortcuts. | Too many tabs = no tabs. You can't find anything in a wall of 40 tabs. |

### Plugins / Extensions Worth Installing

| Plugin | IDE | What It Does |
|---|---|---|
| **GitLens** | VS Code | Supercharged blame, history, and diff. Essentially mandatory. |
| **Error Lens** | VS Code | Shows errors inline instead of requiring hover. |
| **Rainbow Brackets** | Both | Color-codes matching brackets. Helps in deeply nested code. |
| **Key Promoter** | IntelliJ | Shows you the shortcut for every mouse action. Trains muscle memory. |
| **String Manipulation** | IntelliJ | Case conversion, escaping, encoding — things you'd otherwise do in a browser tool. |
| **SonarLint** | Both | Catches bugs and code smells in real time. Pairs with your CI SonarQube gate. |
| **Database Tools** | IntelliJ (built-in) | Query databases, view schemas, export data — without leaving the IDE. |

---

## Common Mistakes

### 1. "I'll Just Use a Text Editor"

Using Vim/Nano/Sublime for a codebase with 500+ files and complex type hierarchies is like using a screwdriver to build a house. Text editors are great for config files, quick edits, and scripting. They are not great for navigating, debugging, and refactoring large codebases.

**Exception**: If you're a proficient Vim/Neovim user with LSP configured, you're essentially running an IDE. This advice is for people who use basic text editors without language server integration.

### 2. "I Know the Shortcuts I Need"

No, you don't. You know the shortcuts you've always used. Install **Key Promoter X** (IntelliJ) or check the keyboard shortcut reference (`Cmd+K Cmd+S` in VS Code) and actively try one new shortcut per week. After 6 months, you'll be dramatically faster.

### 3. "The Debugger Is Too Slow to Set Up"

This is true exactly once — the first time. After that, your run configurations are saved. The time you invest in setting up a debug configuration pays for itself the first time you avoid a 20-minute print-statement debugging session.

### 4. "I Prefer the Terminal for Git"

For `git push` and `git checkout`, sure. For understanding what changed, resolving merge conflicts, and doing partial staging? The IDE is objectively better. Use both — don't handicap yourself with just one.

### 5. "I Don't Need Code Coverage Visualization"

You don't *need* it. But the first time you run coverage and see a bright red line through your "obvious" error handling code, you'll realize that "I'm pretty sure that's tested" is not the same as "I can see that it's tested."

### 6. Using the Mouse for Everything

If you find yourself reaching for the mouse more than the keyboard, you're doing it wrong. The mouse is for visual tasks (drag-and-drop, graphical debugging). Keyboard shortcuts are for everything else. Every mouse action has a keyboard equivalent — learn it.

---

## IDE Features Checklist by Task

### Starting a New Task

- [ ] Pull latest changes
- [ ] Read the ticket/issue — understand what "done" looks like
- [ ] Use **Find Usages** and **Go to Definition** to understand the code you'll change
- [ ] Use **Call Hierarchy** to understand the impact of your changes
- [ ] Set up a **run configuration** if you don't have one

### Writing Code

- [ ] Use **Quick Fix** (`Option+Enter` / `Cmd+.`) to generate boilerplate
- [ ] Use **Live Templates / Snippets** for common patterns (`sout`, `fori`, `psvm` in IntelliJ)
- [ ] Let auto-import handle imports
- [ ] Use **Extract Variable/Method** to keep methods short as you go
- [ ] Run the relevant test after each meaningful change — not after 200 lines

### Debugging

- [ ] Set **conditional breakpoints** for targeted investigation
- [ ] Use **Evaluate Expression** to test hypotheses in-place
- [ ] Use **Step Over** for trusted code, **Step Into** for suspicious code
- [ ] Use **Watches** for values you need to track across multiple breakpoints
- [ ] Check **Log Breakpoints** before adding `println` / `console.log`

### Code Review

For the full code review process (comment conventions, PR sizing, reviewer selection, anti-patterns), see Code Review. The IDE-specific tools that support reviews:

- [ ] Use **IDE diff viewer** instead of the web UI for complex changes
- [ ] Use **Annotate/Blame** to understand the history of changed lines
- [ ] Use **Find Usages** to verify that renamed/moved symbols are updated everywhere
- [ ] Use **Run with Coverage** to check if new code paths are tested

### Investigating a Bug

- [ ] Start with **Exception Breakpoints** if you have a stack trace
- [ ] Use **Find in Path** to search for error messages, status codes, or log strings
- [ ] Use **Local History** to see what changed recently (even uncommitted changes)
- [ ] Use **Call Hierarchy** to trace the execution path to the failure point
- [ ] Use **Annotate/Blame** to find when the bug was introduced

---

## Quick Reference: The Commands That Matter

**Navigation**
- Go to File: `Cmd+Shift+O` / `Cmd+P`
- Go to Symbol: `Cmd+Option+O` / `Cmd+T`
- Go to Definition: `Cmd+B` / `F12`
- Find Usages: `Option+F7` / `Shift+F12`
- Navigate Back/Forward: `Cmd+[` `Cmd+]` / `Ctrl+-` `Ctrl+Shift+-`
- File Structure: `Cmd+F12` / `Cmd+Shift+O`
- Recent Files: `Cmd+E` / `Ctrl+R`

**Editing**
- Quick Fix: `Option+Enter` / `Cmd+.`
- Rename: `Shift+F6` / `F2`
- Extract Method: `Cmd+Option+M`
- Extract Variable: `Cmd+Option+V`
- Multiple Cursors: `Option+Click`
- Select Next Occurrence: `Ctrl+G` / `Cmd+D`
- Comment Line: `Cmd+/`

**Debugging**
- Step Over: `F8` / `F10`
- Step Into: `F7` / `F11`
- Step Out: `Shift+F8` / `Shift+F11`
- Evaluate Expression: `Option+F8` / Debug Console
- Toggle Breakpoint: `Cmd+F8` / `F9`

**Search**
- Search Everywhere: `Shift+Shift` / `Cmd+Shift+P`
- Find in Path: `Cmd+Shift+F`
- Replace in Path: `Cmd+Shift+R`

**Git**
- Commit: `Cmd+K`
- Push: `Cmd+Shift+K`
- Update/Pull: `Cmd+T` (IntelliJ)
- Show Log: `Cmd+9` (IntelliJ)

---

*The best time to learn your IDE was when you started using it. The second best time is today. Pick one shortcut from this guide, use it exclusively for a week, and move to the next.*
