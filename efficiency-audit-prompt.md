# Operating-file efficiency audit

Audit the specified instruction, workflow, context, and memory files.
Keep findings and edits as proposals.

Optimize for the lowest total cost of completing correct work,
including reading, tool calls, verification, and rework.

## Process

1. Establish scope. If no files are specified, identify likely targets
   and ask only when ambiguity would materially change the audit.
2. Read target files and necessary references. Use current instructions
   and narrowly relevant history to assess alignment; label dated or
   unverified evidence.
3. Identify material friction, contradictions, outdated guidance,
   duplication, vague instructions, and missing completion criteria.
4. Recommend the smallest complete changes. Keep shared guidance in one
   authoritative location and place branch-specific detail behind clear
   references. Preserve authorization, data protection, and security.
5. Check that proposals preserve necessary behavior, resolve identified
   conflicts, and introduce no new contradictions. Stop when each
   material finding has a recommendation or an explicit limitation.

## Output

Start with the most critical friction points.

For each finding, give the file and line or section, practical impact,
and proposed deletion, rewrite, or addition.

Then provide revised affected sections. Include full replacement files
only when needed to make the proposal understandable.

Distinguish savings in always-loaded instructions from savings in
occasional reference material. Label estimates and unresolved limits.
