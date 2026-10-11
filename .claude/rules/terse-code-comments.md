# Code comments: what it does, nothing more

A code comment says WHAT the code does, or flags a non-obvious behaviour a
reader must not break. It never carries the reasoning for a change, and it
never uses jargon.

## No change-reasoning

Don't explain WHY a value changed, what it used to be, what problem prompted
it, or what alternative was rejected. That reasoning is already captured — in
the commit message and `CHANGELOG.md`. In a comment it only duplicates them,
and rots at the next change.

- Bad:  `burst 30 was too tight and 503'd real visitors, so 100`
- Bad:  `maxretry above the auth jails — a forbidden hit is a weaker signal`
- Good: `limit_req zone=req_per_ip burst=100 nodelay;   # per-IP request rate cap`

Explaining a non-obvious behaviour is still fine — that is WHAT, not
why-it-changed:

- OK: `# server-level: a location-level add_header would drop these`

## No jargon

Write for someone who knows the tool but not this conversation. No incident
references, no internal shorthand, no coined terms ("green x-bit", "café IP").
Name the mechanism plainly or leave it out.

## Where the rest goes

- Why it changed / what it fixes / what was weighed → commit message.
- What changed, as a factual index → `CHANGELOG.md` (see maintain-changelog).
- Deeper design rationale → a design doc (e.g. `ACL_DESIGN.md`).
