# Worked example: structural FEA interview

What the three modes produce for a structural finite element analysis role.
The candidate profile below is invented for illustration.

## Example candidate

Four years of structural FEA at an aerospace supplier. Abaqus and ANSYS
Mechanical. Projects: titanium bracket fatigue redesign, bolted flange joint
with preload and contact, composite panel buckling. Habit of validating
against hand calculations and test.

## Prep mode output

1. **Question map** generated for the role: meshing and convergence; element
   selection (shell vs solid, integration order, shape functions); boundary
   conditions and over-constraint; loads and inertia relief; linear vs
   nonlinear (contact, plasticity, large deflection); bolted joints and
   pretension; buckling (eigenvalue vs nonlinear); modal, harmonic, random
   vibration; fatigue (S-N, stress concentration, Kt vs Kf); composites and
   failure criteria (Tsai-Wu, Hashin); stress singularities; submodeling and
   symmetry; verification and validation; software specifics; behavioral.
2. **Bank items** extracted from a guided conversation about the three
   projects, plus one talking point per technical topic. Each has an opener
   drafted by Claude from the candidate's details, edited by the candidate,
   and marked rehearsed after two spoken repetitions in mock mode.
3. Typical size for this role: 60–80 items, 3–5 hours of prep.

### Sample bank items

| Kind | Title | Opener (rehearsed) | Keywords | Scaffold |
|---|---|---|---|---|
| story | Bracket fatigue redesign | The closest thing I've done is a titanium bracket that kept failing fatigue at a fillet while the model said it was fine. I rebuilt the mesh locally at the radius and ran a convergence study, and the peak stress moved about 15 percent. We opened the radius and the next article passed test. | fillet hot spot · convergence study · Kt vs hand calc · radius change | STAR |
| technical | Mesh convergence | I treat convergence as a required step, not optional. I refine locally at the hot spot, track peak stress across two or three refinements, and stop when the change is under about 5 percent. If it keeps climbing I suspect a singularity, a sharp corner or a point load, not a real stress. | local refinement · 2–3 passes · <5% change · singularity check | Technical |
| technical | Shell vs solid | Shells when one dimension is thin, roughly 1 to 10 or thinner, and bending dominates. Solids where through-thickness stress matters: a bolted flange, a thick lug, a contact face. If I'm unsure I run both on a small region and compare. | thickness ratio · bending vs through-thickness · contact → solids | Technical |
| technical | Boundary conditions | Over-constraining is the most common mistake I see. I apply restraints the way the part is actually held, use soft springs or an inertia-relief check when fully fixed would be fake, and I always check reaction forces against the applied load. | as-built restraints · over-constraint → fake stiffness · reactions = load | Technical |
| technical | Validation | Before I trust a model I check it three ways: a hand calculation on a simplified section, reactions summing to the applied load, and a deflection in the expected order of magnitude. If any is off, the model is wrong, not the hand calc. | hand calc · reactions = load · deflection sanity | Technical |
| technical | Linear vs nonlinear | I go nonlinear when the physics demands it: contact, plasticity past yield, or large deflection where geometry changes stiffness. Otherwise linear, because it is faster and easier to validate. | contact · plasticity · large deflection · default linear | Technical |
| story | Bolted flange with preload | (body only, no rehearsed opener yet) | pretension step → service load · friction contact · solids at flange · slip and gapping | STAR |

## Live cue cards

**Q1. "Tell me about a time your FEA result disagreed with test data."**
Level 1, clear match, ≈0.5 s.

```
▌ The closest thing I've done is a titanium bracket that kept failing
▌ fatigue at a fillet while the model said it was fine. I rebuilt the
▌ mesh locally at the radius and ran a convergence study, and the peak
▌ stress moved about 15 percent. We opened the radius and the next
▌ article passed test.

  fillet hot spot · convergence study · Kt vs hand calc · radius change
  STAR → Situation · Task · Action · Result
```

**Q2. "How do you make sure your mesh is adequate?"** Level 1.

```
▌ I treat convergence as a required step, not optional. I refine locally
▌ at the hot spot, track peak stress across two or three refinements, and
▌ stop when the change is under about 5 percent. If it keeps climbing I
▌ suspect a singularity, a sharp corner or a point load, not a real stress.

  local refinement · 2–3 passes · <5% change · singularity check
  Technical → what · why · how you check · example from your work
```

**Q3. "Shell or solid elements, and when?"** Level 1.

```
▌ Shells when one dimension is thin, roughly 1 to 10 or thinner, and
▌ bending dominates. Solids where through-thickness stress matters: a
▌ bolted flange, a thick lug, a contact face. If I'm unsure I run both
▌ on a small region and compare.

  thickness ratio · bending vs through-thickness · contact → solids
```

**Q4. "How would you model a bolted joint with preload?"** Level 2, two items
fit and the story has no rehearsed opener, ≈1.2 s.

```
  bolt pretension step → then service load · contact pairs with friction
  solids at the flange · check slip and gapping · compare to VDI / hand calc

  Angle: your bolted-joint project first, then the modelling steps.
```

**Q5. "Explain hourglassing and how you detect it."** Level 3, nothing in
the bank. Scaffold only; the question is logged for drill.

```
  Technical → definition · why it happens · how you detect it · what you do
  (logged for drill)
```

## Mock mode on the same questions

Mock mode is where composed answers exist. For Q5:

```
Interviewer: Explain hourglassing in reduced-integration elements and how
you would detect it.

[candidate answers aloud; transcript scored]

Model answer: Hourglassing is a spurious zero-energy deformation mode in
reduced-integration elements. The element bends in a pattern the single
integration point can't see, so it has no stiffness against it. You spot
it as a zigzag or checkerboard in the deformed mesh and as artificial
strain energy; in Abaqus I compare ALLAE to ALLIE and want it under 5 to
10 percent. Fixes: enhanced hourglass control or fully integrated
elements, finer mesh, and no point loads on a single node.

Feedback: you defined it correctly but skipped detection. Add the energy
check. Structure 4/5, specificity 2/5, evidence 3/5, concision 4/5.
Focus next: name the detection method before the fix.

[Save as opener]  [Drill again tomorrow]
```

Saving turns the model answer into a bank item. The candidate edits it into
their own words, says it twice, and Q5 becomes a level-one card next time.

## What this does and does not do

- With the prep done, most questions in a structural FEA interview land on
  level 1 or 2, and the sentences on screen are ones the candidate has
  already said aloud.
- Without prep, almost everything lands on level 3 and the card is a
  structure hint. The tool does not compose answers to unprepared questions
  during an undisclosed real interview; that is a design decision, not a gap.
