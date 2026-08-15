# Chapter 176b: Parametric Constraint Solvers — Degrees of Freedom, Decomposition, and Numerical Methods

**Target audiences:** Graphics application developers building or evaluating CAD/CAE tooling on Linux; engineers integrating or choosing among open-source and commercial geometric constraint solvers; readers of Ch176 (OpenCASCADE) and Ch176a (State-of-the-Art CAD AI) who need the algorithmic detail those chapters reference but don't cover in depth.

---

## Table of Contents

1. [Scope: Three Kinds of Constraint Solving](#1-scope-three-kinds-of-constraint-solving)
2. [Degrees of Freedom and the Constraint Graph](#2-degrees-of-freedom-and-the-constraint-graph)
   - 2.1 [Counting DOF, Entity by Entity](#21-counting-dof-entity-by-entity)
   - 2.2 [Well-, Under-, and Over-Constrained Systems](#22-well--under--and-over-constrained-systems)
   - 2.3 [Laman's Theorem and Combinatorial Rigidity](#23-lamans-theorem-and-combinatorial-rigidity)
   - 2.4 [Diagnosing Redundancy: The Witness Configuration Method](#24-diagnosing-redundancy-the-witness-configuration-method)
3. [Numerical Solving Methods](#3-numerical-solving-methods)
   - 3.1 [Constraint Satisfaction as Nonlinear Least Squares](#31-constraint-satisfaction-as-nonlinear-least-squares)
   - 3.2 [SolveSpace / libslvs: Symbolic Differentiation and a Minimal C API](#32-solvespace--libslvs-symbolic-differentiation-and-a-minimal-c-api)
   - 3.3 [FreeCAD PlaneGCS: Four Algorithms Over One Jacobian](#33-freecad-planegcs-four-algorithms-over-one-jacobian)
   - 3.4 [KittyCAD ezpz: A Text-Format Solver DSL](#34-kittycad-ezpz-a-text-format-solver-dsl)
4. [Constructive, Symbolic, and Propagation-Based Methods](#4-constructive-symbolic-and-propagation-based-methods)
   - 4.1 [Rule-Based Construction Sequences](#41-rule-based-construction-sequences)
   - 4.2 [Symbolic/Algebraic Methods](#42-symbolicalgebraic-methods)
   - 4.3 [Local Propagation: Sketchpad, DeltaBlue, and the Cassowary Lineage](#43-local-propagation-sketchpad-deltablue-and-the-cassowary-lineage)
5. [Decomposition-Recombination: Solving at Production Scale](#5-decomposition-recombination-solving-at-production-scale)
   - 5.1 [Owen's DOF-Labeled Graph Decomposition](#51-owens-dof-labeled-graph-decomposition)
   - 5.2 [Top-Down vs. Bottom-Up; Dulmage-Mendelsohn and Tarjan SCC](#52-top-down-vs-bottom-up-dulmage-mendelsohn-and-tarjan-scc)
   - 5.3 [Siemens D-Cubed DCM: DR-Planning as a Commercial Product](#53-siemens-d-cubed-dcm-dr-planning-as-a-commercial-product)
6. [The Multiple-Solutions Problem](#6-the-multiple-solutions-problem)
7. [Implementation Survey](#7-implementation-survey)
   - 7.1 [SolveSpace/libslvs (Recap)](#71-solvespacelibslvs-recap)
   - 7.2 [FreeCAD PlaneGCS and Its WASM Port](#72-freecad-planegcs-and-its-wasm-port)
   - 7.3 [KittyCAD ezpz](#73-kittycad-ezpz)
   - 7.4 [CAD Sketcher: Wrapping Rather Than Reimplementing](#74-cad-sketcher-wrapping-rather-than-reimplementing)
   - 7.5 [Siemens D-Cubed and Proprietary In-House Solvers](#75-siemens-d-cubed-and-proprietary-in-house-solvers)
8. [From Solved Sketch to Solid: The Handoff Back to the B-Rep Kernel](#8-from-solved-sketch-to-solid-the-handoff-back-to-the-b-rep-kernel)
   - 8.1 [Sketch Entities Becoming Wires and Profiles](#81-sketch-entities-becoming-wires-and-profiles)
   - 8.2 [3D Assembly (Mate) Constraints as the Same Machinery, Different Vocabulary](#82-3d-assembly-mate-constraints-as-the-same-machinery-different-vocabulary)
9. [Roadmap](#9-roadmap)

[Integrations](#integrations)

---

## 1. Scope: Three Kinds of Constraint Solving

Ch176 covers OpenCASCADE Technology (OCCT) as a boundary-representation (B-rep) modeling kernel: topology (`TopoDS_Shape`), geometry (curves and surfaces), and Boolean operations. OCCT ships no parametric constraint solver of its own — sketch-based parametric CAD applications built on it, notably FreeCAD, embed a separate solver library and feed its output into OCCT's edge/wire/face constructors. That architectural split is not an OCCT-specific gap; it is the norm across the industry. A constraint solver and a B-rep kernel are different pieces of mathematics solving different problems, and this chapter is about the one Ch176 deliberately left unexplored: given a set of geometric primitives and a set of relationships between them, find a consistent assignment of numeric values (point coordinates, radii, angles) that satisfies every relationship simultaneously.

"Constraint solving" is used for at least three distinct problems that share vocabulary but not mathematics:

- **2D sketch constraint solving** — the subject of this chapter. Entities are points, lines, circles, and arcs confined to a plane; constraints are coincidence, parallelism, tangency, and dimensional relationships (distance, angle, radius). The system is generally nonlinear (a tangency constraint between a line and a circle is a quadratic relationship), and it is solved once per sketch edit, interactively, as the user drags a point or types a dimension.
- **3D assembly (mate) constraint solving** — the same DOF-analysis and decomposition machinery generalized from 2D entities (2–3 DOF each) to 3D rigid bodies (6 DOF each), with a different constraint vocabulary (concentric, coincident-face, distance, angle-between-parts). Covered in §8.2.
- **Linear-arithmetic UI layout constraint solving** — Cassowary, the incremental simplex-based solver behind Apple's Auto Layout and used by the Rust `cassowary` crate that Ratatui can select for terminal-UI layout (Ch192). Cassowary constraints are linear equalities and inequalities over box edges (`left + width <= right`), solved by an incremental variant of the simplex method — an entirely different algorithm family from the nonlinear geometric solvers in this chapter, despite the shared word "constraint." [Source](https://constraints.cs.washington.edu/solvers/cassowary-tochi.pdf)

This chapter is scoped to the first bullet in depth, the second in outline (§8.2), and mentions the third only to draw the boundary (§4.3).

## 2. Degrees of Freedom and the Constraint Graph

### 2.1 Counting DOF, Entity by Entity

Every sketch entity contributes some number of free scalar variables — its degrees of freedom — before any constraint is applied:

| Entity | DOF | Free variables |
|---|---|---|
| Point | 2 | x, y |
| Line segment | 4 | endpoint₁ (x, y), endpoint₂ (x, y) |
| Circle | 3 | center (x, y), radius |
| Arc | 5 | center (x, y), radius, start angle, end angle |

Each constraint removes some number of DOF by adding equations that relate these variables:

| Constraint | DOF removed | Equation form |
|---|---|---|
| Coincident (point–point) | 2 | p₁.x = p₂.x, p₁.y = p₂.y |
| Horizontal / Vertical | 1 | p₁.y = p₂.y (or p₁.x = p₂.x) |
| Parallel | 1 | direction cross product = 0 |
| Tangent (line–circle) | 1 | distance(center, line) = radius |
| Distance (dimensional) | 1 | \|p₁ − p₂\| = d |
| Angle (dimensional) | 1 | angle between two directions = θ |

A sketch is a bipartite graph of entities and constraints; solving it is finding the root of the vector-valued equation system formed by every constraint's residual function set to zero simultaneously. A three-point triangle with all three vertices free has 6 DOF (three points × 2). Applying three "distance" constraints (one per side) removes 3 DOF, leaving 3 — the triangle can still translate and rotate rigidly in the plane. This is expected: a sketch is normally under-constrained with exactly 3 residual DOF (2 translation + 1 rotation) unless something pins it to the origin, which is why SolveSpace and FreeCAD both let a sketch settle with 3 DOF remaining without flagging it as an error (Ch176 §11.3).

### 2.2 Well-, Under-, and Over-Constrained Systems

A sketch is classified by comparing the number of independent constraint equations to the number of free variables:

- **Well-constrained**: equations = variables, and the system has a locally unique (up to the 3 rigid-motion DOF) solution. This is the state a CAD user is working toward — every entity's position is fully determined.
- **Under-constrained**: equations < variables. Some entities remain draggable; the solver has a continuum of valid solutions and picks the one nearest the current configuration (§6).
- **Over-constrained / redundant**: equations > variables, but the extra equations are *consistent* with the others — e.g., constraining a rectangle's opposite sides to be both parallel and equal-length is redundant once the other constraints already imply it. Redundant constraints do not make the system unsolvable, but they do make the Jacobian rank-deficient, and most solvers flag them explicitly since removing the wrong one can change designer intent.
- **Conflicting**: equations > variables and the extra equations are inconsistent — e.g., constraining the same line to be both horizontal and vertical. No solution exists.

FreeCAD's Sketcher surfaces this classification directly in its viewport color coding: fully constrained geometry renders green, under-constrained geometry renders white with the specific under-constrained points marked in red, and — notably — over-constrained/redundant geometry is marked in **purple**, with the *specific redundant constraints* (not just the general area) highlighted so the user knows exactly which ones to remove. [Source](https://github.com/FreeCAD/FreeCAD/issues) (Sketcher DOF and redundant-constraint reporting is documented across FreeCAD's issue tracker and wiki; the purple/green/white convention is also visible in the Sketcher UI itself.)

### 2.3 Laman's Theorem and Combinatorial Rigidity

Before running an expensive numerical solve, a solver benefits from knowing — combinatorially, from the constraint graph's structure alone — whether a 2D system is *generically* well-constrained, independent of the actual numeric values involved. This is the domain of **rigidity theory**. The central result for 2D bar-joint frameworks is the **Geiringer–Laman theorem** (Hilda Geiringer, 1927; independently rediscovered by Gerard Laman, 1970): a graph with n vertices and m edges, realized generically in the plane as points connected by fixed-length bars, is *minimally rigid* if and only if m = 2n − 3, and every subgraph on n' vertices has at most 2n' − 3 edges (the Laman condition — no sub-cluster is itself over-braced). [Source](https://en.wikipedia.org/wiki/Geiringer%E2%80%93Laman_theorem)

Two important caveats for CAD practitioners, both drawn from a 2022 review of geometric constraint solving methods:

- Laman's theorem applies to **generic** distance-constraint frameworks. Real sketches mix distance, angle, tangency, and incidence constraints, and can hit non-generic (degenerate) configurations — e.g., three points that happen to be collinear — where the combinatorial count says "rigid" but the specific numeric instance is singular. Production solvers use Laman-style counting as a fast pre-check, not a substitute for the numerical solve.
- **3D rigidity has no equivalent complete combinatorial characterization.** The natural 3D generalization of Laman's count (m = 3n − 6, with a per-subgraph bound) is *necessary but not sufficient* — there exist "generically rigid" bar frameworks by the count that are nonetheless flexible in 3D. The canonical counterexample is the **double banana**: two rigid tetrahedra sharing two vertices with the count satisfied but the whole assembly free to rotate about the shared axis. The **Ortuzar** and related counterexamples show the same failure mode in other configurations. This is why 3D assembly constraint solvers (§8.2) cannot rely on a clean rigidity test the way 2D sketch solvers can, and instead lean more heavily on the numerical/decomposition machinery of §3 and §5. [Source](https://arxiv.org/abs/2202.13795)

### 2.4 Diagnosing Redundancy: The Witness Configuration Method

Knowing that a system is over-constrained (Jacobian rank-deficient) is not the same as knowing *which* constraints are the redundant ones — a rank-deficient Jacobian just says "some combination of rows is linearly dependent," and multiple different subsets could be picked as "the" redundant set. The **Witness Configuration Method (WCM)** solves this by evaluating the constraint system's Jacobian at a randomly perturbed "witness" configuration (rather than only at the current solved configuration) and using rank analysis at that witness point to identify which individual constraint rows are structurally, rather than coincidentally, dependent on others. Because the witness configuration is generic (chosen to avoid accidental numeric degeneracies), rank deficiencies observed there reflect the constraint graph's actual combinatorial redundancy rather than an artifact of the current specific geometry. [Source](https://arxiv.org/abs/2202.13795)

## 3. Numerical Solving Methods

### 3.1 Constraint Satisfaction as Nonlinear Least Squares

Every constraint contributes one or more scalar residual functions r_i(x) that should equal zero at a solution, where x is the vector of all free entity variables. Solving the sketch is finding x such that **F(x) = [r_1(x), r_2(x), ..., r_m(x)]ᵀ = 0**. Because constraints like tangency and distance are quadratic in the underlying coordinates, F is nonlinear, and the standard family of methods is iterative, Newton-type root-finding:

- **Newton–Raphson / Gauss–Newton**: linearize F around the current estimate xₖ using its Jacobian J, solve the linear system J·Δx = −F(xₖ) for the update step, set xₖ₊₁ = xₖ + Δx, and repeat until ‖F‖ falls below a tolerance. This converges quadratically near a solution but can diverge or oscillate far from one, and is undefined if J is singular (exactly the redundant/over-constrained case from §2.2).
- **Levenberg–Marquardt (LM)**: damps the Newton step by adding λI to JᵀJ before solving, blending between a Gauss-Newton step (λ→0, fast near the solution) and a gradient-descent step (λ large, robust far from the solution). λ is adjusted adaptively each iteration based on whether the step actually reduced the residual.
- **Dog-Leg (Powell's dogleg / trust-region)**: computes both the Gauss-Newton step and the steepest-descent step and picks a point along the piecewise-linear path between them that stays inside a trust-region radius, adjusting the radius based on how well the linear model predicted the actual residual reduction. Generally considered more robust than plain LM for badly-conditioned geometric systems, at the cost of slightly more bookkeeping per iteration.
- **BFGS (quasi-Newton)**: builds up an approximation to the Jacobian (or its inverse) from successive gradient evaluations instead of computing it exactly each iteration, trading some convergence speed for avoiding explicit Jacobian assembly — useful when residuals are expensive to differentiate.

Because Newton-type methods only find a *local* root, the initial guess matters enormously — this is the seed of the multiple-solutions problem discussed in §6, and it is why every production solver **warm-starts** from the previously solved configuration rather than an arbitrary or zeroed initial guess.

### 3.2 SolveSpace / libslvs: Symbolic Differentiation and a Minimal C API

Ch176 §11.3 already covers SolveSpace's Newton-Raphson solver and its use of symbolic (not numeric finite-difference) differentiation via an embedded expression system, backed by Eigen for the linear algebra, with a configurable maximum of roughly 1,024–2,048 unknowns. That detail is not restated here; what is worth adding is the shape of the actual embeddable API, `libslvs`, which is what applications like CAD Sketcher (§7.4) bind against rather than reimplementing a solver. The core data structures are plain C structs — a parameter, an entity referencing up to four parameters, and a constraint referencing up to four entities plus a numeric value:

```c
// solvespace/include/slvs.h
typedef struct {
    Slvs_hParam     h;
    Slvs_hGroup     group;
    double          val;
} Slvs_Param;

typedef struct {
    Slvs_hEntity    h;
    Slvs_hGroup     group;
    int             type;
    Slvs_hEntity    wrkpl;
    Slvs_hEntity    point[4];
    Slvs_hEntity    normal;
    Slvs_hEntity    distance;
    Slvs_hParam     param[4];
} Slvs_Entity;

typedef struct {
    Slvs_hConstraint    h;
    Slvs_hGroup         group;
    int                 type;
    Slvs_hEntity        wrkpl;
    double              valA;
    Slvs_hEntity        ptA;
    Slvs_hEntity        ptB;
    Slvs_hEntity        entityA;
    Slvs_hEntity        entityB;
    Slvs_hEntity        entityC;
    Slvs_hEntity        entityD;
    int                 other;
    int                 other2;
} Slvs_Constraint;

#define SLVS_RESULT_OKAY                0
#define SLVS_RESULT_INCONSISTENT        1
#define SLVS_RESULT_DIDNT_CONVERGE      2
#define SLVS_RESULT_TOO_MANY_UNKNOWNS   3
#define SLVS_RESULT_REDUNDANT_OKAY      4

DLL void Slvs_Solve(Slvs_System *sys, uint32_t hg);
```
[Source](https://github.com/solvespace/solvespace/blob/master/include/slvs.h)

`Slvs_Solve` takes a fully-populated system (all params/entities/constraints for a given group handle `hg`) and solves it in place, writing solved values back into the `Slvs_Param.val` fields and reporting one of the four result codes above. `SLVS_RESULT_REDUNDANT_OKAY` is notable: it signals that redundant constraints were detected *and tolerated* (the solve still converged), distinct from `SLVS_RESULT_INCONSISTENT`, which means the redundant constraints actually conflict — directly reflecting the redundant-vs-conflicting distinction from §2.2 in the API's own vocabulary.

### 3.3 FreeCAD PlaneGCS: Four Algorithms Over One Jacobian

FreeCAD's Sketcher does not use SolveSpace's solver; it embeds its own, **PlaneGCS**, forked originally from the wxCAD/gcs.cpp geometric constraint solver lineage and substantially extended since. PlaneGCS exposes the algorithm choice as an explicit enum rather than hard-coding one method, letting the same Jacobian assembly feed four different iterative solvers:

```cpp
// FreeCAD/src/Mod/Sketcher/App/planegcs/GCS.h
enum SolveStatus {
    Success = 0,
    Converged = 1,
    Failed = 2,
    SuccessfulSolutionInvalid = 3
};

enum Algorithm {
    BFGS = 0,
    LevenbergMarquardt = 1,
    DogLeg = 2
};

int solve(bool isFine = true, Algorithm alg = DogLeg, bool isRedundantsolving = false);
```
[Source](https://github.com/FreeCAD/FreeCAD/blob/main/src/Mod/Sketcher/App/planegcs/GCS.h)

Despite `BFGS` being enum value 0, `DogLeg` is the actual default passed to `solve()` — FreeCAD's own experience is that the trust-region Dog-Leg method is the more reliable general-purpose choice for sketch-sized systems, with LM and BFGS available as fallbacks the Sketcher retries automatically if the default fails to converge (`SolveStatus::Failed`), and `SuccessfulSolutionInvalid` covering the case where the numeric solve converges but produces a geometrically degenerate result (e.g., a zero-radius circle). The `isRedundantsolving` flag lets the caller request a second pass that specifically classifies which constraints are the redundant ones (§2.4), rather than only reporting that redundancy exists.

### 3.4 KittyCAD ezpz: A Text-Format Solver DSL

`ezpz` is a newer (2026), MIT-licensed, Rust-implemented constraint solver from KittyCAD, notable for defining problems through a small dedicated text format rather than a graph-construction API — closer to how a symbolic algebra system takes input than how SolveSpace or PlaneGCS are driven:

```
# constraints
point p
point q
p.x = 0
p.y = 0
q.y = 0
vertical(p, q)

# guesses
p roughly (3, 4)
q roughly (5, 6)
```
[Source](https://github.com/KittyCAD/ezpz)

The `# guesses` section is an explicit, first-class part of the problem definition — the DSL makes the Newton-type solver's dependence on an initial estimate (§3.1) visible in the input format itself, rather than leaving it as an implicit solver parameter. As a new, well-funded, permissively-licensed entrant, `ezpz` is a direct competitive data point against decades-old commercial solvers like D-Cubed (§7.5); its longer-term trajectory as a maintained project is worth tracking rather than treating as settled. *Note: needs verification* — public sources describing `ezpz`'s internal iterative method as Gauss-Newton specifically were not independently confirmed against a primary statement in the project's own documentation at the time of writing; treat that specific algorithm attribution as provisional.

## 4. Constructive, Symbolic, and Propagation-Based Methods

Numerical iteration (§3) is not the only way to solve a constraint system, and understanding the alternatives clarifies why iterative Newton-type solving became the dominant production approach.

### 4.1 Rule-Based Construction Sequences

A **constructive** (or rule-based) solver treats the DOF-labeled constraint graph as a pattern-matching problem: it looks for local sub-patterns that a known geometric construction rule can resolve directly — e.g., "two points and a distance constraint between them plus one point's position already known" matches a compass-and-straightedge circle-intersection rule that places the second point exactly, with no iteration. The solver repeatedly finds and applies matching rules, effectively re-deriving a ruler-and-compass construction sequence for the sketch. This is fast (closed-form, no convergence risk) when it works, but its coverage is only as complete as its rule library — constraint combinations that don't match a known pattern (which becomes common as sketches grow more complex, or use constraints like tangency between two free-floating circles) fall back to numerical methods anyway, which is why constructive solving in modern CAD tools is typically a fast-path optimization layered in front of a numerical solver rather than a replacement for one.

### 4.2 Symbolic/Algebraic Methods

Constraint equations can, in principle, be solved exactly using computer algebra: **Gröbner basis** computation reduces the polynomial system to a canonical form from which all solutions can be extracted, and the **Wu–Ritt method** (characteristic-set decomposition) offers an alternative exact-elimination route for polynomial constraint systems. Both are complete — they find every solution, not just the one nearest an initial guess — but both suffer from computational complexity that grows explosively with the number of variables and equations, making them impractical for interactive sketches with dozens to hundreds of constraints. They remain useful for offline analysis (e.g., proving a constraint system's solution count, or verifying a numerical solver against ground truth on small test cases) but are not used as an interactive sketch solver's primary method in any of the tools surveyed in §7. [Source](https://arxiv.org/abs/2202.13795)

### 4.3 Local Propagation: Sketchpad, DeltaBlue, and the Cassowary Lineage

The oldest constraint-solving lineage in interactive graphics is **local propagation**: rather than solving the whole system simultaneously, propagate values through the constraint graph one constraint at a time, using each constraint to compute one of its variables from the others, iterating until the system stabilizes. Ivan Sutherland's **Sketchpad** (1963) — widely regarded as the first interactive constraint-based drawing system — used a one-pass propagation method supplemented by relaxation for cases the one-pass method could not resolve directly. [Source](https://novedge.com/blogs/design-news/design-software-history-from-sketchpad-to-d-cubed)

**DeltaBlue** (Sannella, Maloney, Freeman-Benson, and Borning, 1993) generalized this into an incremental, priority-based local-propagation algorithm for UI constraint systems: constraints are annotated with a strength (required, strong, medium, weak), and DeltaBlue efficiently re-satisfies only the constraints affected by a change rather than re-solving the whole graph, using a spanning-tree-like structure over the constraint graph to decide propagation order. [Source](https://dl.acm.org/doi/pdf/10.1145/76372.77531)

This lineage connects directly to **Cassowary** (Badros, Borning, and Stuckey, 2001), the incremental simplex-based solver for *linear* arithmetic constraints that underlies Apple's Auto Layout and is available to Rust programs — including, per Ch192, Ratatui's layout engine — via the `cassowary` crate. [Source](https://constraints.cs.washington.edu/solvers/cassowary-tochi.pdf) The connection is genealogical and terminological, not mathematical: Cassowary solves linear equalities/inequalities over box-model edges with the simplex method, which is a fundamentally different problem from the nonlinear, generally non-monotonic geometric relationships (tangency, perpendicularity between arbitrary lines) that a 2D sketch solver must handle. A reader coming from UI layout work who recognizes "constraint solver" from Cassowary/Auto Layout should not assume the same algorithm underlies CAD sketch solving — this is the precise boundary the "three kinds" framing in §1 exists to mark.

## 5. Decomposition-Recombination: Solving at Production Scale

### 5.1 Owen's DOF-Labeled Graph Decomposition

A single whole-graph Newton-type solve (§3.1) works well for sketches with tens of constraints, but its cost and — more importantly — its numerical robustness both degrade as sketches scale into the hundreds of constraints that production mechanical CAD sketches routinely contain. The standard production-scale technique is **Decomposition-Recombination (DR-planning)**, introduced by John Owen's 1991 ACM Solid Modeling paper on degrees-of-freedom analysis for geometric constraint solving: the constraint graph is recursively partitioned into small, independently-rigid sub-clusters (a cluster being a subgraph whose internal DOF count matches its internal constraint count exactly, per the DOF bookkeeping in §2.1–2.2), each sub-cluster is solved on its own — by whichever method fits its size, often the exact constructive rules of §4.1 for very small clusters — and the solved sub-clusters are then recombined, treating each as a single rigid unit with its own reduced DOF, and the process repeats one level up the hierarchy. [Source](https://dl.acm.org/doi/10.1145/112515.112566)

### 5.2 Top-Down vs. Bottom-Up; Dulmage-Mendelsohn and Tarjan SCC

DR-planning algorithms split into two complementary strategies depending on which direction of the DOF spectrum (§2.2) they are optimized for:

- **Top-down** approaches start from the whole graph and repeatedly split it at articulation pairs (pairs of vertices whose removal disconnects the graph, or more precisely, whose removal reduces the graph to independent rigid pieces) — well suited to **under-constrained** systems, where the goal is identifying which sub-regions are already rigid versus still free.
- **Bottom-up** approaches start from individual entities and greedily merge them into progressively larger rigid clusters wherever a locally-complete constraint set is found — well suited to **over-constrained** systems, where merging naturally surfaces the redundant constraint set once two already-independently-rigid clusters are found to be connected by more constraints than the 3 (2D) or 6 (3D) needed to fix their relative pose.

The underlying graph algorithms are drawn from classical combinatorial theory applied to the bipartite variable–equation graph: **Dulmage–Mendelsohn decomposition** partitions a bipartite graph into an ordered sequence of subsets reflecting over-determined, well-determined, and under-determined regions of a matching, and **Tarjan's strongly-connected-components algorithm** identifies the maximal tightly-coupled sub-blocks within the well-determined region — together giving a DR-planner a structural map of which constraint clusters must be solved jointly and which can be solved independently and merged. [Source](https://arxiv.org/abs/2202.13795)

### 5.3 Siemens D-Cubed DCM: DR-Planning as a Commercial Product

**Siemens D-Cubed 2D DCM** and **3D DCM** are the commercial embodiments of this technique most widely deployed in the CAD industry — D-Cubed has historically been licensed into SolidWorks and is the constraint engine behind Onshape's sketcher, among others. Siemens' own product documentation describes 2D DCM's degrees-of-freedom analysis and diagnostic reporting (identifying redundant and conflicting constraints, directly corresponding to §2.2/§2.4) and dedicated handling for spline-curve constraints (constraining a NURBS control point or tangent-continuity relationship, which sits outside the point/line/circle/arc vocabulary this chapter has otherwise used) as first-class DCM features rather than bolt-ons. [Source](https://www.siemens.com/en-us/products/plm-components/d-cubed/2d-dcm/) D-Cubed's internal solving algorithms are not publicly documented at the source-code level (it is closed-source, licensed as an embeddable component) — its public-facing material describes DR-planning-style decomposition and DOF diagnostics conceptually, consistent with Owen's technique, but the chapter does not claim more architectural detail than Siemens' own published material supports.

## 6. The Multiple-Solutions Problem

A well-constrained system (§2.2) has a *locally* unique solution, but geometric constraint systems routinely admit **multiple distinct globally valid solutions**. Two circles constrained tangent to a third can be tangent from either side; a linkage constrained by a set of distance constraints can often be folded into a mirror configuration that satisfies every equation identically. Newton-type methods (§3.1) converge to *whichever* root is nearest their starting point — they have no mechanism to enumerate or choose between branches on their own.

There is an exhaustive answer: **homotopy continuation** methods can, in principle, enumerate all real (and complex) roots of a polynomial constraint system by continuously deforming from a system with known solutions to the target system while tracking every solution path. This gives completeness that Newton-Raphson alone cannot, but at a computational cost that makes it impractical for interactive, per-keystroke sketch solving — it belongs in the same "exhaustive but offline" category as the symbolic methods of §4.2. [Source](https://arxiv.org/abs/2202.13795)

The practical answer used by every interactive solver surveyed in §7 is **warm-starting**: seed each new solve with the previously solved configuration (or, for a newly-added entity, with the constraint's own "guess" hint, as in `ezpz`'s explicit `# guesses` block from §3.4) rather than an arbitrary or zeroed starting point. Because Newton-type iteration converges to the nearest root, warm-starting means that as a user drags a point continuously, the solver tracks the same solution branch continuously — the tangent circle stays on the same side, the linkage doesn't unexpectedly flip — turning branch selection from a mathematical ambiguity into a predictable, UX-driving consequence of the solver's own iteration mechanics. This is a case where an algorithmic implementation detail (which root a damped-Newton iterate converges to) is directly responsible for a user-facing interaction guarantee (dragging feels continuous, not jumpy) that has no separate explicit implementation — it falls out of §3.1's methods for free, provided the solver is disciplined about always seeding from the prior state.

## 7. Implementation Survey

This section indexes the concrete solvers referenced throughout the chapter against each other; it does not restate the algorithmic detail already given in §3 and Ch176 §11.3, or duplicate Ch176a's CAD-tool comparison table.

### 7.1 SolveSpace/libslvs (Recap)

Covered in depth in Ch176 §11.3 (Newton-Raphson with symbolic differentiation, Eigen-backed, ~1,024–2,048 max unknowns) and in §3.2 above (the `libslvs` C API and its redundant-vs-inconsistent result codes). GPLv3-licensed. [Source](https://github.com/solvespace/solvespace)

### 7.2 FreeCAD PlaneGCS and Its WASM Port

PlaneGCS (§3.3) is FreeCAD's native C++ Sketcher solver, LGPL-licensed as part of the FreeCAD source tree. It has been independently ported to WebAssembly by the **Salusoft89/planegcs** project, which compiles the same C++ solver core to WASM and wraps it with a TypeScript API for in-browser use — the same reuse pattern Ch176 §11.5 documents for `opencascade.js` wrapping OCCT itself, meaning a browser-based sketch tool can now embed both the B-rep kernel and the constraint solver that feeds it via the identical compile-to-WASM strategy, independently applied by two separate projects to two separate C++ codebases. [Source](https://github.com/Salusoft89/planegcs)

### 7.3 KittyCAD ezpz

Covered in §3.4. MIT-licensed, Rust. As a 2026-era new entrant with a permissive license, it is a direct competitive counterpoint to both the copyleft SolveSpace solver and the closed-source commercial D-Cubed (§7.5) — worth tracking for whether it gets embedded by other open tools the way SolveSpace's solver already has been (§7.4). [Source](https://github.com/KittyCAD/ezpz)

### 7.4 CAD Sketcher: Wrapping Rather Than Reimplementing

**CAD Sketcher** is a Blender addon that adds parametric 2D sketch constraint solving to Blender's modeling workflow. Rather than implementing its own solver, it wraps SolveSpace's Python bindings (`pyslvs`/SolveSpace's own Python module) directly, translating Blender's mesh/curve data into SolveSpace's entity/constraint model and reading solved values back. This is a useful concrete case study in an application-layer choice covered more generally in Ch244 (Blender AI/MCP integration): reusing an external, already-mature solver via its scripting API rather than reimplementing constraint-solving logic inside the host application, trading some integration friction for avoiding an enormous engineering investment. [Source](https://github.com/hlorus/CAD_Sketcher)

### 7.5 Siemens D-Cubed and Proprietary In-House Solvers

Siemens D-Cubed 2D/3D DCM (§5.3) remains the dominant proprietary constraint solver licensed into third-party commercial CAD products. Current-generation solver internals for other major commercial CAD packages — specifically, whatever in-house sketch solver Autodesk Fusion uses internally, and Dassault Systèmes SolidWorks' own historical and current solver stack beyond its documented past use of D-Cubed — are **not independently corroborated by a primary source available at the time of writing** and are deliberately not characterized here beyond that D-Cubed licensing history. *Note: needs verification* if a future edition has access to primary vendor documentation on these specific internals.

## 8. From Solved Sketch to Solid: The Handoff Back to the B-Rep Kernel

### 8.1 Sketch Entities Becoming Wires and Profiles

A solved 2D sketch is, on its own, just a set of numerically-satisfied point coordinates — it becomes a 3D solid only once it re-enters the B-rep kernel as topology. In the FreeCAD architecture, this is the literal boundary between the Sketcher workbench (owns PlaneGCS, produces solved 2D geometry) and the Part Design workbench (consumes that geometry as a closed wire/profile and calls OCCT operations — `BRepBuilderAPI_MakeEdge` per solved line/arc segment, chained into a wire, then extrude or revolve to produce a solid). SolveSpace's own internal pipeline performs the analogous handoff without a workbench boundary, since sketch-solving and solid-generation (extrude/revolve/loft) are both native to the same application, but the underlying seam — numerically-solved 2D entities being reinterpreted as B-rep edges before any solid operation can run — is the same in both tools. This is precisely the seam Ch176's own Roadmap flags as never having been unified inside OCCT/OCAF itself: OCCT has no native concept of a "constraint," only geometry and topology, so every application built on it must own this translation step itself, using whichever solver (or none, for direct/explicit modeling workflows) it chooses.

### 8.2 3D Assembly (Mate) Constraints as the Same Machinery, Different Vocabulary

Everything in §2 through §6 generalizes from 2D sketch entities to 3D assembly components with two changes: each rigid body contributes 6 DOF (3 translation + 3 rotation, versus a 2D point's 2) instead of 2–5, and the constraint vocabulary shifts to assembly-level relationships — concentric (two cylindrical faces share an axis), coincident (two planar faces touch), distance and angle between parts, and so on — instead of point/line/circle sketch constraints. The DOF-counting bookkeeping of §2.1, the well-/under-/over-constrained classification of §2.2, and the DR-planning decomposition of §5 all carry over structurally; what does *not* carry over cleanly is the 2D rigidity theory of §2.3 — as noted there, 3D rigidity has no complete Laman-style combinatorial test, so 3D assembly solvers lean more heavily on the numerical (§3) and decomposition (§5) machinery and less on a fast combinatorial pre-check. This is also why 3D DCM is offered by Siemens as a distinct product from 2D DCM (§5.3) rather than the same engine with a bigger DOF count — the practical algorithm mix differs, not just the entity vocabulary.

## 9. Roadmap

**Near-term (6–12 months):**
- `ezpz`'s continued development as a well-funded, MIT-licensed entrant competing directly with decades-old commercial DCM-class solvers — worth tracking whether it gets embedded by other open-source CAD tools the way SolveSpace's solver already has been by CAD Sketcher. [Source](https://github.com/KittyCAD/ezpz)
- Continued WASM porting of native C++ solvers (PlaneGCS via Salusoft89, alongside `opencascade.js`'s existing OCCT port, Ch176 §11.5) broadening solver reuse across native, browser, and addon (Blender) contexts without separate reimplementation.

**Medium-term (1–3 years):**
- Whether any open-source solver closes the practical gap with D-Cubed on large-assembly (hundreds to thousands of mates) DR-planning performance and redundancy diagnostics remains an open question; no public benchmark comparing `ezpz`, PlaneGCS, and 3D DCM at that scale was located at the time of writing.

**Long-term:**
- The incompleteness of 3D rigidity theory (§2.3) is a genuine open mathematical problem, not merely an engineering gap — any future combinatorial 3D rigidity test that closes it (or proves it cannot be closed generally) would directly change how 3D assembly solvers can pre-validate constraint systems before running numerical iteration.
- Whether OCCT/OCAF ever gains a first-party constraint-solving layer, unifying the sketch-solver/B-rep-kernel seam described in §8.1, remains tracked as an open item in Ch176's own Roadmap and is not restated as a new prediction here.

## Integrations

- **Ch176 (OpenCASCADE Technology)**: the architectural reason OCCT ships no solver, and the source of the SolveSpace/libslvs Newton-Raphson detail this chapter builds on in §3.2 and §7.1 without restating; §8.1 elaborates the sketch→B-rep handoff Ch176's Roadmap flags as unresolved.
- **Ch176a (State-of-the-Art CAD AI)**: SketchGen/Vitruvion-style constraint-graph generative models operate directly on the DOF/constraint-graph representation defined in §2 of this chapter; `ezpz`'s row in Ch176a's CAD-tool comparison table is expanded on here in §3.4 and §7.3.
- **Ch192 (Ratatui)**: Cassowary, covered there as Ratatui's optional linear-arithmetic layout engine, is the terminologically-adjacent but mathematically distinct cousin of the nonlinear geometric solvers this chapter covers — see §4.3 for the explicit boundary.
- **Ch244 (Blender AI/MCP)**: CAD Sketcher's choice to wrap SolveSpace's own Python module (§7.4) rather than reimplement a solver is a parallel case of the external-tool-wrapping pattern Ch244 covers for Blender scripting/MCP integration.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
