# Venu 3 Claude Coding Knowledge Base

A collection of Markdown documents capturing real-world lessons learned while building Garmin Connect IQ apps for the **Venu 3** in Monkey C — lessons that are not in the official SDK documentation.

These files were produced collaboratively between Myself and Claude (Anthropic) through iterative trial and error: write code, test in the simulator, deploy to the real watch, observe what broke, document the root cause and fix. Nothing is added until it is confirmed working on real hardware or in the simulator.

This is still a relatively new project and the amount of information these files contain is limited. I will be adding to this continuously - essentially I'm just vibe coding with Claude. He's doing all the coding and I'm having him add to these documents every time we run into something he couldn't just automatically do correctly. The idea is that the more of these knowledge base files I generate, the more skilled Claude will become. You can of course provide these to whatever model you want - I'm interested in getting local models to do this as well. But I'm sticking to Claude for actually adding to the knowledge base.

## What's in This Repository

| File | Contents |
|------|----------|
| `INDEX.md` | Master index — section tables and key facts for every file. **Start here.** |
| `garmin_connectiq_knowledge_base.md` | Core SDK reference: device specs, API overview, permission table, drawing, input, manifest |
| `garmin_developer_key_guide.md` | Generating and configuring the DER-format RSA key required for device deployment |
| `garmin_development_addendum.md` | Practical workflow: manifest pitfalls, project structure, install/uninstall, simulator setup |
| `garmin_sdk_sample_code_patterns.md` | Copy-paste code patterns extracted from SDK samples and real projects |
| `monkeyc_analyzer_unreachable_statement_guide.md` | Root cause and permanent solutions for "Statement is not reachable" compiler warnings |
| `venu3_practical_lessons.md` | Device-specific knowledge: data screen configuration, bezel constraints, layout guidelines |
| `steps_and_time_development_lessons.md` | Data field: daily step count + time of day |
| `consolidated_field_development_lessons.md` | Data field: 6-metric dual-triangle layout, indoor step/distance, obscurity flags |
| `timerhr_field_development_lessons.md` | Data field: HR zone coloring, negative y-coordinates, font sizing |
| `hrv_logger_development_lessons.md` | Widget: beat-to-beat HRV intervals, ActivityRecording session requirement |
| `skin_temp_widget_development_lessons.md` | Widget: glance view styling, AMOLED calibration, persistence layer |
| `customhrv_development_lessons.md` | Watch app: touchscreen input, swipe direction, storage schema, arrow rendering |

All documents follow a consistent format with a numbered section index and a "Related Knowledge Base Documents" section linking to other relevant files.

---

## Who This Is For

Anyone who wants to use an LLM (Claude, GPT-4, Gemini, etc.) as a coding assistant for Garmin Connect IQ development on the Venu 3 — and wants the LLM to start with real-world knowledge rather than having to rediscover every non-obvious quirk from scratch.

The files are device-specific (Venu 3, SDK 8.4.1) but most lessons apply broadly to other round Garmin devices and recent SDK versions.

---

## How to Use These Files with an LLM

### What You Need

1. **These knowledge base files** — clone or download this repository
2. **The Garmin Connect IQ SDK** — download free from the [Garmin developer portal](https://developer.garmin.com/connect-iq/sdk/)
   - The SDK contains API documentation (`doc/Toybox/`), sample projects (`samples/`), and device specifications
   - Install it and note the path — you will reference it in your instructions to the LLM
3. **VS Code** with the [Monkey C extension](https://marketplace.visualstudio.com/items?itemName=garmin.monkey-c)
4. **SDK Consolidation Script** *(optional but recommended)* — [github.com/MJenkinsonGIT/SDKConsolidationScript](https://github.com/MJenkinsonGIT/SDKConsolidationScript)
   - Converts the SDK's HTML documentation into clean LLM-friendly Markdown
   - Produces a single structured reference you can load selectively into a model's context
   - Supports device-specific builds that strip irrelevant content and inject inline warnings for APIs not supported on your target device
   - Without this, you would need to feed raw SDK HTML files into your LLM — workable but significantly noisier

### Providing Context to the LLM

The LLM needs two things to be an effective coding partner:

**1. The knowledge base files**
Feed all `.md` files from this repository into the LLM's context. How you do this depends on the platform:
- **Claude Projects (recommended):** Add all `.md` files as Project Knowledge. Claude will search them automatically when answering questions.
- **Other platforms:** Paste file contents into the system prompt, or upload files if the platform supports it. Prioritise `INDEX.md` and `garmin_connectiq_knowledge_base.md` if context space is limited.

**2. Access to the SDK**
Give the LLM a tool that can read files from your local filesystem (e.g. Claude's computer use, a file-reading MCP server, or Cursor/Copilot workspace access). Point it at your SDK installation. The LLM should read API docs and sample code directly rather than guessing.

If the LLM cannot access your filesystem, manually copy relevant SDK doc pages or sample code into the conversation when working on a specific feature.

### Instructions to Give the LLM

Provide a project instruction file (sometimes called a system prompt, project instructions, or `CLAUDE.md` depending on your platform). A template is included at the bottom of this README — copy it, fill in your paths, and use it as your starting instructions.

---

## Development Workflow

This is the iterative cycle that produced these knowledge files:

```
1. Identify a gap in Garmin's built-in functionality
        ↓
2. Find the relevant SDK docs and sample code
        ↓
3. Write the initial implementation with the LLM
        ↓
4. Test in the Connect IQ Simulator (F5 in VS Code)
        ↓
5. Build for device (Ctrl+F5) → copy .prg to GARMIN\APPS\ on the watch
        ↓
6. Test on the real watch during an actual activity
        ↓
7. Note what broke → diagnose root cause with the LLM → fix
        ↓
8. When something non-obvious is confirmed working, document it
        ↓
9. Update the relevant knowledge base file(s) and INDEX.md
        ↓
        Repeat
```

### Key Principles

- **Test on real hardware.** The simulator misses important things: AMOLED brightness, bezel clipping, sensor behaviour, backlight interactions.
- **Document root causes, not just fixes.** "Use X instead of Y" is less useful than "Use X instead of Y because Z."
- **Don't document until confirmed.** Unverified information actively harms future sessions.
- **Use percentage-based layout.** The same data field renders at full-screen size and at 1/4 slot size. Fixed pixel positions break in one context or the other.

---

## Contributing / Extending This Knowledge Base

If you add new lessons while using these files, please follow the existing format so the files remain useful as LLM context:

- Each file has a `## Numbered Index` section and uses `## N. Section Title` numbered headings
- Each file ends with a `## Related Knowledge Base Documents` section
- `INDEX.md` has an entry for every file with a section table and key facts summary
- Nothing gets added until it is tested and confirmed

---

## Sample Project Instruction File

Copy the template below, replace the placeholder paths with your actual paths, and use it as your project instructions / system prompt when starting a coding session with an LLM.

```markdown
# Project Instructions

## SDK Reference

Always consult the SDK before writing code or making implementation decisions:

- **SDK root:** [PATH TO YOUR SDK INSTALLATION]
  Example (Windows): C:\Users\YourName\AppData\Roaming\Garmin\ConnectIQ\Sdks\connectiq-sdk-win-8.4.1-2026-02-03-e9f77eeaa
- **API docs:** [SDK root]\doc\Toybox\
- **Sample apps:** [SDK root]\samples\

Do not rely on general coding knowledge when the SDK or samples have the answer.
Only fall back to general reasoning when the SDK documentation is silent on a topic.

SDK files are read-only. Never edit them. If an SDK file needs to be customised,
copy it into the working directory first and edit the copy.

---

## Working Directory

All app code, resources, and project files live here:

[PATH TO YOUR WORKING DIRECTORY]
Example: C:\Users\YourName\AppData\Roaming\Garmin\ConnectIQ\Sdks\[SDK VERSION]\Claude Coding Docs\

Create sub-folders per project as needed. Read, create, and edit files freely within this directory.

---

## Knowledge Base

The knowledge base is located at:

[PATH TO YOUR KNOWLEDGE BASE]
Example: [Working Directory]\Venu3ClaudeCodingKnowledge\

It captures real-world lessons that are not in the official SDK documentation.
Keeping it accurate and up to date is critical — if a chat session ends unexpectedly,
the knowledge base is the only record of what was learned.

### The Golden Rule

Do not add anything to the knowledge base until it has been tested and confirmed
working — on the simulator, on the real device, or both as appropriate.

### When to Update the Knowledge Base

Update when any of the following are confirmed working:
- A non-obvious API behaviour discovered through testing
- A bug and its root cause and fix
- A layout value or pixel measurement arrived at through iteration
- A compiler warning pattern and its solution
- Any finding that contradicts or corrects something already in a file

### How to Update the Knowledge Base

1. **Choose the right file.** Check INDEX.md before creating a new file.
   Add to an existing file if the lesson fits naturally there.

2. **Match the existing format exactly.** Every file follows this structure:
   - Title block (project name, SDK version, app type, status)
   - Intro paragraph stating what the file covers
   - `## Numbered Index` section linking to all `## N. Section Title` headings
   - Sections using `## N. Section Title` with `### Subsection` headings inside
   - Code blocks with `monkeyc` syntax highlighting
   - `## Related Knowledge Base Documents` section (bulleted, one-line description per file)
   - Footer: `*Last updated: [Month Year] — SDK [version]*`

3. **If adding a new section to an existing file:**
   - Add a numbered entry to that file's `## Numbered Index`
   - Add the section before the Related Documents section
   - Update the footer date

4. **If creating a new file:**
   - Follow the full format above
   - Use naming convention: [project_name]_development_lessons.md
   - Add a full entry to INDEX.md
   - Add cross-references to related files' `## Related Knowledge Base Documents` sections

5. **Always update INDEX.md** after any knowledge base change.

6. **Update cross-references** in any related files if the new content is relevant to them.

---

## Development Approach

This is an iterative learning project. The typical cycle is:

1. Write or modify code
2. Test in the Connect IQ Simulator (F5 in VS Code)
3. Build and deploy to the physical device (Ctrl+F5 → copy .prg to GARMIN\APPS\)
4. Test on the watch during an actual activity
5. Note what worked and what didn't
6. Update the knowledge base with confirmed findings
7. Repeat

Document failures as well as successes where the failure illuminates why
something works the way it does.
```

---

*Device: Garmin Venu 3 · SDK: 8.4.1 / API Level 5.2 · Updated: February 2026*
