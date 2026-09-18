# Concepts

Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as ce-compound and ce-compound-refresh process learnings; direct edits are fine. Glossary only, not a spec or catch-all.

Each heading carries in parentheses the Italian name used by the plans and handoffs written before this translation.

## Work and verification

### Implementation unit (unita' d'implementazione)
The smallest piece of work the plan assigns that integrates on its own, identified by a code and tracked by an issue.

Units declare their dependencies, and the order between two of them can be binding rather than merely advised: when the defects two units fix mask each other, fixing one alone produces a visible deterioration that reads as a regression. A unit that is implemented and integrated is not yet an accepted unit: it still lacks bench verification.

### Reconnaissance (ricognizione)
The fact-finding pass run on the unmodified client, before writing code, to confirm that the defects the analysis predicts actually show up.

It has stop conditions declared in advance: if the expected symptom does not appear, the analysis is revised before implementation instead of going ahead. A stop condition is valid only if its outcome changes depending on whether the analysis is right or wrong; conditions phrased in terms of what is seen on screen often lack this property.

### Bench verification (verifica al banco)
The verification carried out by the operator with the radio on, as distinct from the checks that run without hardware.

It is the condition that closes a unit, and nothing substitutes for it: checks without hardware can say that the code does what the code says, not that the station works. It follows that a unit can stay integrated and not accepted for a long time.

## Client artefacts

### Portable variant (variante portabile)
The self-contained artefact of the client: a single document with style and code embedded, which can be copied to a USB stick and opened in a browser with no server and no build. It is the most useful property of the project, and it binds every unit to stay inside that file.

### Split variant (variante scomposta)
The same application with style and code in separate files, meant to be served from a website.

It is derived from the portable variant, not parallel to it: it is regenerated, never edited by hand. The generation tool works in both directions, so a change made to the split variant is erased without warning at the next regeneration.

## Spectrum

### Spectrum source (sorgente spettro)
The stream that feeds the spectrum trace. There are two: the IQ samples, on which the client computes the transform locally, and the already-decimated bins computed by the server.

There is only one drawn trace, and it does not know which of the two feeds it; the choice is explicit, not inferred, because the operator knows better than any heuristic whether they are on a local network or on a thin link. The two sources differ in bandwidth by more than an order of magnitude, and that is why the second one exists.

### Trace liveness (liveness della traccia)
The property that distinguishes a trace that is being updated from one that is merely drawn.

It does not coincide with data availability: a trace can stay visible indefinitely showing the last content received, if the flag that allows drawing it is not invalidated by the same condition that produces it. The reliable discriminator is a frame counter that advances between two readings, never the drawing state.
