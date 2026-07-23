# CLAUDE.md

## Project Overview
This repository is an Obsidian vault for organizing Unreal Engine framework knowledge.
The goal is to write markdown (.md) documents summarizing Unreal Engine code syntax and related concepts,
and to generate a graph visualizing relationships between documents using Obsidian's Canvas feature.

## Language Rules
- This CLAUDE.md file is always written in English.
- When summarizing `source/` content into `framework/` markdown documents, write primarily in Korean, but keep technical terms in English.

## Directory Structure
```
root/
├── CLAUDE.md
├── source/                      ← directory for unrefined, raw documents
├── framework/                   ← directory for cleaned up and summarized documents; md files here are used to generate the graph
│   ├── <FunctionName>.md        ← function without `::`
│   └── <ClassName>/
│       └── <Name>.md            ← function written as `Class::Name`
└── chapter/
    └── core/
        └── <ChapterName>.canvas ← one canvas per chapter; nodes reference framework/ md files
```
- Follow the flow of cleaning up and summarizing documents from `source/` and moving them into `framework/`.
- Canvas graph nodes are generated only from .md documents in `framework/`.
- Canvas files live under `chapter/core/`, named after the chapter.

## Scope of Work
- Only perform the requested task. Do not perform additional work that wasn't instructed (refactoring, file reorganization, style unification, etc.) on your own.
- If the scope of work is ambiguous, ask first rather than expanding the scope arbitrarily.

## Rules for Modifications
- When modifying an existing .md or .canvas file, always explain "why" the modification is being made first.
- Do not modify a file without explaining the reason.
- Keep explanations concise (do not write unnecessarily long descriptions).

## Response Style
- All responses should be short and direct (terse).
- Do not add unnecessary elaboration, greetings, or repeated summaries.

## Document Writing Rules
- Markdown documents should focus on Unreal Engine code syntax and related explanations.
- Link related concepts between documents using Obsidian link syntax (`[[document name]]`).

## Page Template (`framework/`)
Every document created in `framework/` follows this template exactly:

````markdown
---
related:
  - "[[FunctionA]]"
  - "[[FunctionB]]"
tags:
  - <SourceFile>_cpp
---

```cpp
<source code>
```

## 설명
- <explanation of the code>
````

- `related` and `tags` are Obsidian frontmatter properties, not body text, so they stay hidden in
  canvas node previews (see the `canvas-hide-properties.css` snippet).
- `related` is a list property of Obsidian links. Quote each value so the link is not parsed as YAML.
  For a function written as `Class::Name`, link as `"[[Class/Class.Name|Class::Name]]"`.
- `related` is always bidirectional. If document A lists B, then B must list A as well.
  After creating or updating a document, add the reverse link to the `related` property of every
  document it points at. Adding a reverse link is the one change allowed on an otherwise skipped document.
- `tags` holds the source file tag (`.cpp` → `_cpp`, `.h` → `_h`, e.g. `LaunchWindows_cpp`).
  Write it as a list and without the `#` prefix — Obsidian's `tags` property adds the `#` itself.
- No H1 title in the body. The file name is the title, so the body starts directly with the source code block.
- `## 설명` holds the explanation of the code. Omit the section if there is nothing to explain.

## Obsidian Snippets
- `.obsidian/snippets/canvas-hide-properties.css` hides the frontmatter block inside canvas node
  previews only, so a card shows just the source code and its explanation.
- Enable it in Settings → Appearance → CSS snippets.

## Classification Rules (`framework/`)
Documents are classified using the `Foundation - <ChapterName> - <EntryName>(<SourceFile>)` marker found in `source/`.
Do not write this marker itself into the resulting document.

`<EntryName>` is either a function name or a type declaration (`class Name`, `struct Name`, `enum Name`).
Both kinds use the same page template; only the file path and the canvas placement differ.

### 1. Function name
- If the function name is written as `Class::Name`, create the file at `framework/<ClassName>/<Class>.<Name>.md`.
  - `:` is not allowed in Windows file names, so `::` is replaced with `.` in the file name
    (`FEngineLoop::PreInit` → `framework/FEngineLoop/FEngineLoop.PreInit.md`).
  - When linking to such a document, restore the original notation with an alias:
    `[[FEngineLoop/FEngineLoop.PreInit|FEngineLoop::PreInit]]`.
- If the name has no `::`, create the file at `framework/<FunctionName>.md`.
- If the function name is `BEGIN`, the document becomes the root node of the canvas named after its chapter.
  - The md file name is the actual source function name (not `BEGIN`).

### 2. Type name (`class` / `struct` / `enum`)
- If `<EntryName>` starts with `class`, `struct`, or `enum`, drop the keyword and use the remaining `<Name>`.
  Either way the document uses the same page template as a function.
- `class` / `struct` → create the file at `framework/<Name>/<Name>.md`.
  - `class FEngineLoop` → `framework/FEngineLoop/FEngineLoop.md`.
  - This is the same folder that holds the type's member function documents, so a type and its members
    stay together (`framework/FEngineLoop/FEngineLoop.PreInit.md`).
- `enum` → create the file at `framework/Enum/<Name>.md`.
  - `enum EWorldType` → `framework/Enum/EWorldType.md`.
  - Enums have no member functions, so they are collected in one shared `Enum` folder instead of
    getting a folder each. Link them as `"[[Enum/EWorldType|EWorldType]]"`.

### 3. Chapter name
- Each document becomes a node in the canvas named after its chapter (`chapter/core/<ChapterName>.canvas`).
- If a canvas of the same name already exists, add the node to it and connect it to related nodes.

### 4. Source file
- The source file may be a `.cpp` or a `.h`; both are handled the same way.
- Tag each document with its source file so documents from the same file can be related.
  The tag goes in the `tags` frontmatter property, with the extension separated by `_`
  (`LaunchWindows.cpp` → `LaunchWindows_cpp`, `LaunchEngineLoop.h` → `LaunchEngineLoop_h`).

## Canvas (Graph) Generation Rules
- Canvas files (.canvas) follow the Obsidian Canvas JSON spec.
- Nodes are created referencing the relevant .md documents.
- Only create edges between nodes when there is an actual conceptual relationship between the documents.
- Every time a document is added to `framework/`, it must also be added as a node to
  `chapter/core/<ChapterName>.canvas`, where `<ChapterName>` comes from the
  `Foundation - <ChapterName> - <EntryName>(<SourceFile>)` marker used to classify it.
  - If the canvas does not exist yet, create it.
  - Node `file` paths point to the md document in `framework/` (`file` type nodes).

### Function cluster
- The `BEGIN` document is the root node of its chapter canvas; other function nodes are laid out
  following the call flow starting from that root.

### Type cluster (`class` / `struct` / `enum`)
- Type documents form their own cluster, laid out separately from the function call-flow cluster
  and never connected to it.
- Place the type cluster to the right of the function cluster, leaving at least one grid step
  (520px) of empty space between the two clusters.
- Only connect types that are actually related to each other (inheritance, containment, a struct or
  enum used by a class). Types with no such relationship stay unconnected.

### Node layout defaults
- Default node size: `width: 460`, `height: 360`.
- Keep a 60px gap between nodes, so the layout grid step is 520 horizontally and 420 vertically.
- Lay out nodes top-down following the call flow, with the root node at the top.
- Nodes on the same call depth share a row and are centered under their caller.

## Incremental Processing
Each document in `source/` carries a `summarize` checkbox property in its frontmatter:

```yaml
---
summarize: true
---
```

1. If `summarize` is `true`, the document is already organized — skip it entirely without parsing.
   If the property is missing or `false`, process the document.
2. When processing, resolve every `Foundation - <ChapterName> - <EntryName>(<SourceFile>)` marker
   to its destination path under `framework/`, list the existing files there once, and compare.
   - Destination exists and the marker adds nothing new → skip. Do not read, rewrite, or reformat it,
     except to add a reverse `related` link pointing back from it.
   - Destination exists but the marker carries content the document does not have yet → merge
     (see Merging into an Existing Document). Never overwrite.
   - Destination missing → create it from the page template.
3. Add only the newly created documents as nodes to `chapter/core/<ChapterName>.canvas`.
   Leave existing nodes, their coordinates, and existing edges untouched.
4. When every marker in the document has been handled, set `summarize: true` in its frontmatter.
   This is the only edit allowed on files in `source/`.
5. Report the result as counts (e.g. `created: 3, skipped: 6`), not as a per-file listing.

To force a document to be organized again, set `summarize` back to `false`.
Re-processing an already existing `framework/` document still requires an explicit request
(e.g. "rewrite", "reapply the template").

## Merging into an Existing Document
When a new entry resolves to a file name that already exists in `framework/`, add to that document
instead of replacing it. Overwriting an existing document is never the default.

1. Read the existing document first, then add only the parts it does not already have.
2. Source code: append the new code as an additional ```cpp block after the existing block(s),
   in the order the entries appear in `source/`. Do not edit or delete the existing block.
3. `## 설명`: append new bullets to the existing list. Do not rewrite or reorder existing bullets.
4. `related` and `tags`: merge as a union — keep the existing values in place and append only the
   new ones. No duplicates.
5. Never delete, reorder, or reformat content that is already in the document.
6. Do not add a second canvas node for the merged entry; the existing node already covers it.
7. If the new content contradicts what the document already states, do not choose silently —
   keep the existing content and report the conflict.
