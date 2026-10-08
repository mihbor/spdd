---
name: spdd-generate
description: Generate code from a structured SPDD prompt file following the REASONS Canvas methodology
---

Generate implementation code from a structured SPDD (Structured Prompt-Driven Development) prompt file, strictly following the Operations sequence and coding norms defined in the prompt.

**Input**: The argument after `/spdd-generate` is the path to the structured prompt file (e.g., `@spdd/prompt/GGQPA-XXX-202602271430-Feat-api-create-agent-endpoint.md`).

**Steps**

1. **If no input provided, ask for the prompt file**

   Use the **AskUserQuestion tool** to ask:
   > "Please provide the path to the structured prompt file (e.g., `@spdd/prompt/xxx.md`)."

   **IMPORTANT**: Do NOT proceed without a valid prompt file path.

2. **Read and parse the structured prompt file**

   Read the prompt file and extract the REASONS Canvas sections:

   | Section | Purpose | Usage |
      |---------|---------|-------|
   | **R** - Requirements | Overall goal and DoD | Understand the business context |
   | **E** - Entities | Domain model and relationships | Reference for class design |
   | **A** - Approach | Implementation strategy | Guide architectural decisions |
   | **S** - Structure | Components and dependencies | Verify layering and relationships |
   | **O** - Operations | Concrete implementation tasks | **Execute in defined order** |
   | **N** - Norms | Engineering standards | Apply to all generated code |
   | **S** - Safeguards | Non-negotiable constraints | Enforce strictly |

   **IMPORTANT**: Read the ENTIRE file carefully. Each section provides critical guidance.

3. **Record the implementation start in `spdd/done.md`**

   As soon as the prompt file is known (before generating code), append a
   section to the implementation log (create the file with the header from
   the existing log if it does not exist yet):

   ```
   ## <STORY-ID>

   - Prompt: `<prompt filename>`
   - Impl start: <now, local time>
   - Completed:
   - Elapsed:
   - Commit:
   - Notes:
   ```

   Derive the story ID from the prompt filename. Record Impl start as the
   actual current local time at the moment generation is invoked — read the
   clock; never estimate it and never copy timestamps from analysis/prompt
   filenames or neighboring log rows, which belong to earlier SPDD phases.
   Leave the remaining fields empty until step 8. Never backfill completion
   at this stage: the section stays open until the result is verified.

4. **Analyze project context**

   Before generating code:
    - Identify the project's technology stack (e.g., Spring Boot, Java version)
    - Locate existing similar patterns in the codebase for reference
    - Identify the correct package structure and directory layout
    - Check for existing base classes, utilities, or configurations to reuse

   **IMPORTANT**: Generated code MUST align with existing project conventions.

5. **Validate the Operations sequence**

   Review the **Operations** section to verify:

   a. **Dependency order is correct**:
    - Classes with no dependencies come first (enums, constants)
    - Classes depend only on previously defined classes
    - No circular dependencies exist

   b. **Task decomposition is complete**:
    - Each operation is atomic and testable
    - No logical gaps between operations
    - All components mentioned in Structure are covered

   c. **Consistency with Structure section**:
    - Inheritance relationships match
    - Dependencies match
    - Layered architecture is respected

   **If issues are found**: Report to user and suggest prompt modifications before proceeding.

   **IMPORTANT**: Do NOT re-plan the sequence. The Operations order is the designed execution order from the Abstraction phase.

6. **Generate code following Operations sequence**

   For each operation in the **Operations** section (in order):

   a. **Read the operation specification**:
    - Responsibility: What the component does
    - Attributes/Methods: Exact fields and signatures
    - Annotations: Required annotations
    - Validation rules: Bean validation or custom logic
    - Business logic: Step-by-step implementation details

   b. **Apply Norms**:
    - Framework metadata and annotation standards appropriate to the detected project
    - Dependency injection or equivalent wiring style, when applicable
    - Exception handling patterns
    - Logging conventions
    - Response format standards

   c. **Enforce Safeguards**:
    - Field validation constraints
    - HTTP status code requirements
    - **Exact error messages** (do not modify)
    - Security constraints
    - Data integrity rules

   d. **Generate the code**:
    - Use correct package path based on project structure
    - Include all required imports
    - Implement exact method signatures as specified
    - Follow the exact validation messages from Safeguards

   **IMPORTANT**:
    - Do NOT deviate from the specifications in Operations
    - Do NOT add features or methods not specified
    - Do NOT change error messages from Safeguards
    - DO reference existing project patterns for consistency

7. **Batch validation after generation**

   After ALL code is generated, perform unified validation:

   a. **Compilation and lint check**:
    - Run the project's compile/build check and any applicable linter to detect syntax and type errors
    - Verify all imports are correct
    - Fix any type mismatches

   b. **Acceptance Criteria verification**:
    - Cross-check with the **Acceptance Criteria Traceability** table (if present)
    - Ensure each AC is addressed by the implementation
    - Verify specified error codes, HTTP status codes, and messages match exactly when applicable

   c. **Structure verification**:
    - Verify the architecture boundaries declared in the prompt are respected
    - Confirm dependency injection or equivalent wiring is correct when applicable
    - Check interface/implementation relationships

   d. **Fix any issues found**:
    - Fix compilation errors
    - Correct import statements
    - Ensure code follows project formatting standards

8. **Report generation summary and record completion**

   Provide a summary to the user:
    - List of created files with their responsibilities
    - Any deviations or assumptions made
    - Validation results (pass/fail for each check)

   Then close the section opened in step 3 of `spdd/done.md`: fill
   Completed with the verification time (when the acceptance check passed —
   headset confirmation, test run, or equivalent — never the code-written
   time), Elapsed as finish minus start, Commit with the first line of the
   INTENDED commit message — craft it now (concise, conventional-commit
   format, e.g. "feat(ai): implement AI city founding for STORY-021-001" —
   never use the hash), and Notes with the verification outcome.

   **Commit ordering (avoids amend/follow-up commits)**: the Commit field
   is filled BEFORE any commit is created. When the user requests a commit,
   stage `spdd/done.md` together with the prompt and code changes and
   create ONE commit using that exact recorded message. Never commit first
   and fill the Commit field afterwards — that ordering forces an amend or
   an extra commit to fix the log.

**Review & Iteration Loop**

If issues are discovered after generation (during testing or code review), follow the SPDD principle:

> **"When reality diverges, fix the prompt first — then update the code."**

1. **Identify the issue**: What behavior is incorrect or missing?

2. **Trace to prompt section**: Which part of the prompt caused this?
    - Wrong requirement interpretation → Update **Requirements**
    - Missing entity/relationship → Update **Entities**
    - Flawed strategy → Update **Approach**
    - Incorrect component design → Update **Structure**
    - Wrong implementation detail → Update **Operations**
    - Missing standard → Update **Norms**
    - Missing constraint → Update **Safeguards**

3. **Update the prompt first**: Modify the relevant section in the prompt file

4. **Regenerate affected code**: Only regenerate the components affected by the prompt change

5. **Commit together when requested**: If the user explicitly requests a commit, commit the updated prompt and code together (plus the `spdd/done.md` log entry recording that exact message) to maintain traceability; otherwise leave the changes for user review

**Example iteration**:
```
Issue: "AgentService interface shouldn't contain business logic"

1. Trace: Operations section defines AgentService with logic
2. Update prompt: Move logic to AgentServiceImpl in Operations
3. Regenerate: Only regenerate AgentService and AgentServiceImpl
4. If requested, commit the prompt change and code change together; otherwise leave them for review
```

**Output**

- All generated source files following the project structure
- Summary of created files and their responsibilities
- Validation results
- Any issues requiring prompt modification

**Guardrails**

- Do NOT generate code without first reading the entire prompt file
- Do NOT re-plan the Operations sequence — execute in the defined order
- Do NOT skip any operation defined in the Operations section
- Do NOT change method signatures, field names, or error messages from the specification
- Do NOT add extra features, endpoints, or fields not specified
- Do NOT patch code directly when issues are found — update prompt first
- Always use the exact error messages from Safeguards
- Always follow Norms for coding style and patterns
- Always verify against Acceptance Criteria after generation
- Always open the `spdd/done.md` section at invocation (step 3) and close it only after verification (step 8); never mark complete at code-written time
- Always record the intended commit message in the `spdd/done.md` Commit field BEFORE creating any commit, and stage `spdd/done.md` in that same commit; never commit first and backfill the log afterwards (no amend, no extra log-only commit)
- Always check for and fix linter errors after batch generation
- When the user explicitly requests a commit, commit prompt and code changes together; otherwise do not create a commit automatically

**Workflow context**

`/spdd-generate` is Phase 3 of the canonical SPDD pipeline. It consumes the structured prompt, validates the requested implementation, and verifies the resulting code; the shared lifecycle and prompt-first iteration rules are defined in the root `AGENTS.md`.

The structured prompt is the **contract** between design and implementation; keep it synchronized with the code throughout the lifecycle.
