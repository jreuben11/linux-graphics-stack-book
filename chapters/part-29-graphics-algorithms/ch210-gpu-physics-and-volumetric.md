# Chapter 210: GPU Geometry Algorithms — Physics Simulation and Volumetric Methods (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers implementing rigid body, fluid, cloth, or fracture simulation pipelines; graphics engineers building SDF-based collision, volumetric reconstruction, or ray marching systems on Linux using Vulkan compute.

This chapter is the third volume of the GPU Geometry Algorithms series. It covers two large domains: Category VI (Physics and Simulation) covers GPU fluid simulation (SPH, FLIP, Eulerian grids), deformable body FEM and projective dynamics, rigid body broadphase and narrowphase, crowd simulation, fracture and destruction, rope, cloth, fluid-solid coupling, granular DEM, hydraulic erosion, fluid surface extraction, continuous collision detection (CCD), and Minkowski sum / motion planning; Category VII (SDF and Volumetric Methods) covers level set evolution, swept volumes, TSDF fusion for 3D reconstruction, SDF-based collision detection, isosurface extraction (Marching Cubes / Dual Contouring), GPU ambient occlusion, volumetric ray marching, and sparse narrow-band level-set GPU solvers.

## Table of Contents

- **[VI. Physics and Simulation](#vi-physics-and-simulation)**
  - [48. Fluid Simulation on the GPU](#48-fluid-simulation-on-the-gpu)
  - [49. Deformable Bodies and Continuum Mechanics](#49-deformable-bodies-and-continuum-mechanics)
  - [50. Rigid Body Dynamics on GPU](#50-rigid-body-dynamics-on-gpu)
  - [51. Crowd and Multi-Agent Simulation](#51-crowd-and-multi-agent-simulation)
  - [52. Fracture and Destruction](#52-fracture-and-destruction)
  - [53. Rope and Cable Simulation](#53-rope-and-cable-simulation)
  - [54. GPU Cloth Simulation](#54-gpu-cloth-simulation)
  - [55. GPU Fluid-Solid Coupling](#55-gpu-fluid-solid-coupling)
  - [56. Granular Material Simulation (DEM)](#56-granular-material-simulation-dem)
  - [57. Terrain Sculpting and Hydraulic Erosion](#57-terrain-sculpting-and-hydraulic-erosion)
  - [58. GPU Fluid Surface Extraction](#58-gpu-fluid-surface-extraction)
  - [59. Continuous Collision Detection (CCD)](#59-continuous-collision-detection-ccd)
  - [60. Minkowski Sum and GPU Motion Planning](#60-minkowski-sum-and-gpu-motion-planning)
- **[VII. SDF and Volumetric Methods](#vii-sdf-and-volumetric-methods)**
  - [61. Level Set Methods](#61-level-set-methods)
  - [62. Swept Volumes and Minkowski Sums](#62-swept-volumes-and-minkowski-sums)
  - [63. Volumetric 3D Reconstruction (TSDF Fusion)](#63-volumetric-3d-reconstruction-tsdf-fusion)
  - [64. SDF-Based Collision Detection](#64-sdf-based-collision-detection)
  - [65. Isosurface Extraction](#65-isosurface-extraction)
  - [66. GPU Ambient Occlusion](#66-gpu-ambient-occlusion)
  - [67. Volumetric Ray Marching](#67-volumetric-ray-marching)
  - [68. GPU Sparse Narrow-Band Level-Set Evolution](#68-gpu-sparse-narrow-band-level-set-evolution)


---

## VI. Physics and Simulation

GPU simulation covers the full range of physical phenomena relevant to real-time and offline rendering: rigid body dynamics with parallel broad-phase collision detection (BVH, spatial hashing), continuous collision detection for fast-moving objects, fluid simulation via SPH particle methods and Eulerian pressure-projection grids, position-based and projective dynamics for cloth and soft bodies, the Discrete Element Method (DEM) for granular materials, and fracture/destruction via Voronoi decomposition. The Minkowski sum and swept volumes appear here because they are the geometric primitives underlying GJK and EPA collision algorithms. All systems run as compute dispatches that write updated vertex positions consumed by the rendering pipeline each frame.

### 48. Fluid Simulation on the GPU

*Audience: graphics application developers, systems developers.*

Fluid simulation divides into two major families: **Eulerian** (fixed grid, track field values at cell centers/faces) and **Lagrangian** (particles follow the fluid). Hybrid schemes (FLIP, APIC) combine both. GPU implementations exploit the data-parallelism in per-cell and per-particle updates, with the bottleneck usually being the pressure Poisson solve.

### 48.1 MAC Grid and Semi-Lagrangian Advection

The **Marker-and-Cell (MAC) grid** stores pressure at cell centers and velocity components at face centers (staggered). For an N³ grid a Vulkan compute layout maps naturally: one thread per cell for pressure, one thread per axis-face pair for velocity.

**Semi-Lagrangian advection** traces each grid point backward along the velocity field for one time step and samples the field at the traced position:

```glsl
// advect.comp
layout(set=0,binding=0) uniform sampler3D velX, velY, velZ;
layout(set=0,binding=1, r32f) uniform writeonly image3D outField;
layout(set=0,binding=2) uniform sampler3D srcField;
layout(push_constant) uniform PC { float dt; float invDx; vec3 origin; } pc;
layout(local_size_x=8, local_size_y=8, local_size_z=8) in;

void main() {
    ivec3 cell = ivec3(gl_GlobalInvocationID);
    vec3  pos  = vec3(cell) + 0.5;            // cell center in grid coords
    vec3  vel  = vec3(texture(velX, pos*pc.invDx).r,
                      texture(velY, pos*pc.invDx).r,
                      texture(velZ, pos*pc.invDx).r);
    vec3  back = (pos - vel * pc.dt) * pc.invDx;  // back-trace
    imageStore(outField, cell, textureLod(srcField, back, 0));
}
```

Third-order BFECC (Back and Forth Error Compensation and Correction) halves the dissipation with two additional advection passes and an error correction term. [Source: Kim et al. "Advections With Significantly Reduced Diffusion and Diffusion", SCA 2005]

### 48.2 Pressure Projection via Conjugate Gradient

After advection the velocity field is not divergence-free. The **pressure Poisson solve** enforces incompressibility:

```
∇²p = (1/Δt) ∇·u*     (Poisson equation)
u   = u* − Δt ∇p       (divergence-free correction)
```

On a uniform grid ∇²p is a sparse symmetric positive-definite matrix amenable to **conjugate gradient (CG)** with an incomplete-Cholesky (IC) or Jacobi preconditioner. Each CG iteration is two sparse matrix-vector products (SpMV) and a few dot products — all O(N³) parallel reductions:

```glsl
// jacobi_iter.comp — one Gauss-Seidel/Jacobi step for ∇²p = rhs
layout(set=0,binding=0, r32f) uniform image3D pressure;
layout(set=0,binding=1) uniform sampler3D rhs;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;

void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    float r  = texelFetch(rhs, c, 0).r;
    float p  = (imageLoad(pressure, c+ivec3(1,0,0)).r +
                imageLoad(pressure, c-ivec3(1,0,0)).r +
                imageLoad(pressure, c+ivec3(0,1,0)).r +
                imageLoad(pressure, c-ivec3(0,1,0)).r +
                imageLoad(pressure, c+ivec3(0,0,1)).r +
                imageLoad(pressure, c-ivec3(0,0,1)).r - r) / 6.0;
    imageStore(pressure, c, vec4(p));
}
```

For production, replace Jacobi with a **multigrid V-cycle**: restrict residual to a coarser grid, smooth there, then prolongate the correction back. Each level halves the grid in each dimension; total work is O(N³) rather than O(N³ log N). [Source: Fedkiw, Stam, Jensen "Visual Simulation of Smoke", SIGGRAPH 2001; McAdams et al. "A Parallel Multigrid Poisson Solver", SCA 2010]

### 48.3 Position-Based Fluids (PBF) on GPU

**Position-Based Fluids** (Macklin & Müller 2013) treats each particle as a density constraint rather than a force, solving for position corrections that keep density at ρ₀:

```
C_i(x) = ρ_i / ρ_0 − 1 = 0
ρ_i = Σ_j m_j W(||x_i − x_j||, h)   // SPH density kernel
```

Per-iteration Lagrange multiplier λᵢ = −Cᵢ / (Σ_k |∇_{x_k} Cᵢ|² + ε):

```glsl
// pbf_lambda.comp
layout(set=0,binding=0) readonly  buffer Pos     { vec4 pos[]; };
layout(set=0,binding=1) readonly  buffer Cells   { int grid[]; };  // spatial hash
layout(set=0,binding=2) writeonly buffer Lambda  { float lambda[]; };

float W_poly6(float r, float h) {
    float x = max(0.0, h*h - r*r);
    return 315.0 / (64.0 * 3.14159 * pow(h,9)) * x*x*x;
}
vec3  W_spiky_grad(vec3 r, float h) {
    float rlen = length(r);
    if (rlen < 1e-6 || rlen > h) return vec3(0);
    float x = h - rlen;
    return -45.0 / (3.14159 * pow(h,6)) * x*x * normalize(r);
}
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    // accumulate density and gradient norm from neighbor cells
    float rho = 0.0, gradNorm = 0.0;
    // (neighbor iteration over spatial hash cells omitted for brevity)
    float Ci = rho / RHO0 - 1.0;
    lambda[i] = -Ci / (gradNorm + EPSILON_RELAXATION);
}
```

After computing λ, position corrections Δpᵢ = (1/ρ₀) Σⱼ (λᵢ + λⱼ) ∇W accumulate via a scatter pass (atomic floats or separate reduction). Surface tension, vorticity confinement, and XPBD compliance extend PBF without structural changes to the GPU pipeline. [Source: Macklin & Müller "Position Based Fluids", SIGGRAPH 2013; Macklin et al. "Unified Particle Physics for Real-Time Applications", SIGGRAPH 2014]

### 48.4 FLIP: Hybrid Particle-Grid

**FLIP** (Fluid Implicit Particles) transfers velocity between particles and a MAC grid each step, combining the detail of Lagrangian particles with the stability of an Eulerian pressure solve:

```
P2G (splat):   u_grid[cell] += w_ip * u_particle[i]   // weight by trilinear kernel
Pressure solve: u_grid = project(u_grid)
G2P (gather):  u_particle[i] += α * (u_grid_new[cell] - u_grid_old[cell])
               (α=0.95: mostly FLIP, 5% PIC damping to suppress noise)
Position update: x_particle += dt * u_particle
```

The P2G scatter pass uses `atomicAdd` on float SSBOs (requires `VK_EXT_shader_atomic_float`). The G2P gather pass reads from the post-solve grid and is fully parallel per particle.

The key GPU dispatch sequence per frame:

```
vkCmdDispatch(P2G_pipeline,    ceil(N_particles/64), 1, 1)
// pressure solve dispatches (CG iterations) …
vkCmdDispatch(G2P_pipeline,    ceil(N_particles/64), 1, 1)
vkCmdDispatch(advect_pipeline, ceil(N_grid/512),     1, 1)
```

[Source: Bridson "Fluid Simulation for Computer Graphics" §7; Zhu & Bridson "Animating Sand as a Fluid", SIGGRAPH 2005]

### 48.5 Lattice Boltzmann Method (LBM)

LBM replaces the Navier-Stokes equations with a kinetic model on a regular lattice (D2Q9 in 2D, D3Q19 or D3Q27 in 3D). Each cell stores 19 distribution functions fᵢ that represent particle populations moving in 19 directions. One step = **collision** (relax toward equilibrium) + **stream** (shift each fᵢ to its neighbor):

```glsl
// lbm_step.comp — BGK collision + stream (D2Q9 shown for brevity)
layout(set=0,binding=0) buffer F    { float f[9 * NCELLS]; };
layout(set=0,binding=1) buffer FNew { float fn[9 * NCELLS]; };
layout(local_size_x=64) in;

const vec2 e[9] = vec2[9](vec2(0,0),vec2(1,0),vec2(-1,0),
                            vec2(0,1),vec2(0,-1),vec2(1,1),
                            vec2(-1,1),vec2(1,-1),vec2(-1,-1));
const float w[9] = float[9](4./9.,1./9.,1./9.,1./9.,1./9.,
                              1./36.,1./36.,1./36.,1./36.);

void main() {
    uint c = gl_GlobalInvocationID.x;
    ivec2 xy = ivec2(c % WIDTH, c / WIDTH);

    // Compute macroscopic density and velocity
    float rho = 0.0; vec2 u = vec2(0);
    for (int i = 0; i < 9; i++) { float fi = f[i*NCELLS+c]; rho += fi; u += fi*e[i]; }
    u /= rho;

    float omega = 1.0 / (3.0*VISCOSITY + 0.5);  // relaxation: tau = 1/omega
    for (int i = 0; i < 9; i++) {
        float eu  = dot(e[i], u);
        float feq = w[i] * rho * (1.0 + 3.0*eu + 4.5*eu*eu - 1.5*dot(u,u));
        float fi  = f[i*NCELLS+c];
        // Stream: write to neighbor cell
        ivec2 dst = xy + ivec2(e[i]);
        if (dst.x >= 0 && dst.x < WIDTH && dst.y >= 0 && dst.y < HEIGHT)
            fn[i * NCELLS + dst.y*WIDTH+dst.x] = fi + omega*(feq - fi);
    }
}
```

LBM is embarrassingly parallel: every cell updates independently. It handles complex boundary conditions naturally and couples to GPU rendering by writing velocity magnitude as a texture. [Source: Körner et al. "Lattice Boltzmann Simulation of Flow in Anisotropic Porous Media", 2016; Lehmann "Esoteric-Pull and Esoteric-Push: An Two-in-One Single-Step Streaming Scheme", Computation 2022]

---

### 49. Deformable Bodies and Continuum Mechanics

*Audience: graphics application developers, systems developers.*

PBD cloth (§39.6) handles thin surfaces via distance and bending constraints. Volumetric deformable bodies require a continuum mechanics formulation — FEM for stiff materials, MPM for materials that fracture, flow, or undergo large deformation. Both decompose naturally into per-element parallel local steps and a global linear solve.

### 49.1 Mass-Spring Systems on GPU

The simplest volumetric deformable body discretises the mesh into a spring network: each edge becomes a damped spring with rest length ℓ₀, stiffness k, and damping d. Forces integrate via explicit Verlet:

```glsl
// spring_force.comp
layout(set=0,binding=0) readonly buffer Pos  { vec4 pos[]; };
layout(set=0,binding=1) readonly buffer Vel  { vec4 vel[]; };
layout(set=0,binding=2) readonly buffer Edges{ uvec2 edges[]; float rest[]; };
layout(set=0,binding=3) buffer Force { vec4 force[]; };
layout(local_size_x=64) in;

void main() {
    uint e    = gl_GlobalInvocationID.x;
    uvec2 ij  = edges[e];
    vec3  d   = pos[ij.y].xyz - pos[ij.x].xyz;
    float len = length(d);
    vec3  axis = d / max(len, 1e-6);
    float fs  = SPRING_K * (len - rest[e]);
    vec3  fd  = DAMPING  * dot(vel[ij.y].xyz - vel[ij.x].xyz, axis) * axis;
    vec3  f   = (fs + length(fd)) * axis;
    atomicAdd(force[ij.x].x,  f.x);   // VK_EXT_shader_atomic_float
    atomicAdd(force[ij.y].x, -f.x);
    // (y,z components similarly)
}
```

Stiff springs require implicit integration (backward Euler, sparse linear solve) to avoid instability at large time steps.

### 49.2 Corotational FEM on GPU

**Corotational FEM** factors out rigid-body rotation from the deformation so the elastic energy is measured relative to the rotated rest configuration. For a tetrahedral mesh:

1. **Per-tet deformation gradient** F = ∂x/∂X (3×3 matrix, one per tet — fully parallel).
2. **Polar decomposition** F = RS (R rotation, S symmetric stretch). Computed via iterative SVD or QR.
3. **Corotational elastic force** on each tet: f = −k_vol · Rᵀ(F − R)

The polar decomposition for 3×3 matrices converges in 4–6 iterations of the cyclic Jacobi method, parallelisable per tet:

```glsl
// corot_tet.comp
layout(local_size_x=64) in;
layout(set=0,binding=0) readonly buffer RestPos { vec3 X[]; };
layout(set=0,binding=1) readonly buffer CurPos  { vec3 x[]; };
layout(set=0,binding=2) readonly buffer Tets    { uvec4 tets[]; };
layout(set=0,binding=3) buffer Force { vec4 f[]; };

mat3 polarRotation(mat3 F) {
    mat3 R = mat3(1.0);
    for (int iter = 0; iter < 6; iter++) {
        vec3 omega = (cross(R[0],F[0])+cross(R[1],F[1])+cross(R[2],F[2])) /
                     (abs(dot(R[0],F[0])+dot(R[1],F[1])+dot(R[2],F[2]))+1e-6);
        float w = length(omega);
        if (w < 1e-8) break;
        R = mat3(cos(w)*R + (1-cos(w))/w/w*outerProd(omega,omega)*R +
                 sin(w)/w*cross(omega, R));  // Rodrigues update
    }
    return R;
}

void main() {
    uint t = gl_GlobalInvocationID.x;
    uvec4 idx = tets[t];
    mat3 Ds = mat3(x[idx.y]-x[idx.x], x[idx.z]-x[idx.x], x[idx.w]-x[idx.x]);
    mat3 Dm = mat3(X[idx.y]-X[idx.x], X[idx.z]-X[idx.x], X[idx.w]-X[idx.x]);
    mat3 F  = Ds * inverse(Dm);
    mat3 R  = polarRotation(F);
    mat3 P  = LAMBDA * (F-R);  // first Piola-Kirchhoff (simplified)
    // distribute forces to vertices via scatter (atomicAdd)
}
```

[Source: Müller & Gross "Interactive Virtual Materials", GI 2004; Irving et al. "Invertible Finite Elements for Robust Simulation", SCA 2004]

### 49.3 Material Point Method (MPM) on GPU

MPM represents material as particles carrying mass, velocity, and deformation gradient, and transfers momentum through a background grid each step. The GPU pipeline alternates between particle-centric and grid-centric passes:

```
P2G (transfer to grid):
    for each particle p:
        for each nearby grid node i:
            grid_mass[i]     += w_ip * mass_p
            grid_momentum[i] += w_ip * mass_p * (vel_p + B_p * (xi - xp))
                                 // APIC affine momentum term
Grid update:
    grid_vel[i] = grid_momentum[i] / grid_mass[i]
    apply gravity, boundary conditions
G2P (transfer back to particles):
    for each particle p:
        vel_p = Σ_i w_ip * grid_vel[i]
        x_p  += dt * vel_p
        F_p   = (I + dt * Σ_i grid_vel[i] * ∇w_ip^T) * F_p
```

The cubic B-spline weight function wᵢₚ = N(xᵢ − xₚ/dx) covers a 3×3×3 stencil, giving each particle 27 grid-node interactions. P2G uses `atomicAdd` on grid mass and momentum (VK_EXT_shader_atomic_float); G2P is a fully parallel gather. MPM handles snow compaction, sand, and fracture by choosing the appropriate constitutive model for the deformation gradient update. [Source: Stomakhin et al. "A Material Point Method for Snow Simulation", SIGGRAPH 2013; Hu et al. "A Moving Least Squares Material Point Method", SIGGRAPH 2018]

---

### 50. Rigid Body Dynamics on GPU

*Audience: graphics application developers, systems developers.*

PhysX 5 (§42 Library Landscape Table B) provides a production GPU rigid body solver; this section covers the algorithms so implementors can build Vulkan-native physics or understand what PhysX does under the hood.

### 50.1 Broadphase: SAP Pair Generation

Sweep-and-prune (SAP) sorts AABB extents along one axis; overlapping intervals on that axis are candidate pairs, checked against the other two axes:

```glsl
// sap_pairs.comp — overlapping AABB pairs from sorted X-extents
layout(set=0,binding=0) readonly buffer SortedMin { float minX[]; uint objID[]; };
layout(set=0,binding=1) readonly buffer MaxX      { float maxX[]; };
layout(set=0,binding=2) buffer Pairs     { uvec2 pairs[]; };
layout(set=0,binding=3) buffer PairCount { uint count; };
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    uint a=objID[i];
    for (uint j=i+1; j<N_OBJECTS && minX[j]<=maxX[a]; j++) {
        uint b=objID[j];
        if (aabbOverlap(aabb[a], aabb[b])) {
            uint slot=atomicAdd(count,1u);
            pairs[slot]=uvec2(a,b);
        }
    }
}
```

Pairs feed the island detection and narrowphase (§24.4 SAT/GJK). [Source: Tonge "Iterative Rigid Body Simulation", GDC 2013]

### 50.2 Island Detection via GPU Label Propagation

Rigid body simulation islands (connected components of the contact graph) must be solved together. Parallel label propagation over active contact pairs:

```glsl
// island_label.comp
layout(set=0,binding=0) readonly buffer Contacts { uvec2 contacts[]; };
layout(set=0,binding=1) buffer Labels  { uint label[]; };
layout(set=0,binding=2) buffer Changed { uint changed; };
layout(local_size_x=64) in;
void main() {
    uint c=gl_GlobalInvocationID.x;
    uint a=contacts[c].x, b=contacts[c].y;
    uint lm=min(label[a],label[b]);
    if (label[a]!=lm) { atomicMin(label[a],lm); atomicOr(changed,1u); }
    if (label[b]!=lm) { atomicMin(label[b],lm); atomicOr(changed,1u); }
}
```

After convergence, islands with disjoint label values dispatch independently in parallel.

### 50.3 Sequential Impulse Solver with Graph Colouring

The sequential impulse (SI) solver applies corrective velocity impulses at each contact. GPU parallelism requires contacts sharing no body to run in the same pass — enforced by graph colouring (4–8 colours suffice for most scenes):

```glsl
// si_solve.comp — one colour pass
layout(set=0,binding=0) buffer Contacts { ContactData c[]; };
layout(set=0,binding=1) buffer BodyVel  { BodyVelocity vel[]; };
layout(local_size_x=64) in;
void main() {
    uint ci=gl_GlobalInvocationID.x;
    ContactData co=c[ci];
    vec3 vrel=linearVelAtPoint(vel[co.bodyA],co.rA)
             -linearVelAtPoint(vel[co.bodyB],co.rB);
    float vn=dot(vrel,co.normal);
    float dL=-(vn+co.bias)/co.effectiveMass;
    dL=max(co.lambda+dL,0.0)-co.lambda;
    c[ci].lambda+=dL;
    applyImpulse(vel[co.bodyA],+dL,co.normal,co.rA,co.invMassA,co.invInertiaA);
    applyImpulse(vel[co.bodyB],-dL,co.normal,co.rB,co.invMassB,co.invInertiaB);
}
```

Run 4–20 iterations; each iteration dispatches one kernel per colour. [Source: Catto "Iterative Dynamics with Temporal Coherence", GDC 2005; Tonge "Solving Rigid Body Contacts", GDC 2013]

### 50.4 Featherstone Articulated Body Algorithm

For articulated bodies (ragdolls, robot arms) the Articulated Body Algorithm (Featherstone 1987) propagates joint quantities in three O(n)-per-chain passes. Independent chains (multiple ragdolls) execute in parallel; joints within one chain are serialised by their topological depth:

```glsl
// aba_pass1.comp — root-to-leaf velocity propagation (one joint per thread)
layout(local_size_x=64) in;
void main() {
    uint j=gl_GlobalInvocationID.x;
    uint p=parent[j];
    if (p==INVALID) { v[j]=externalVel[j]; return; }
    // 6D spatial velocity: v[j] = X[j]*v[p] + S[j]*qdot[j]
    v[j]=spatialTransform(X[j],v[p]) + motionSubspace[j]*qdot[j];
}
// Pass 2 (leaves→root): articulated inertia recursion (reverse topological)
// Pass 3 (root→leaves): acceleration propagation
// Each pass is one dispatch with a barrier before the next.
```

[Source: Featherstone "Rigid Body Dynamics Algorithms", 2008; GPU implementation in NVIDIA PhysX 5 articulation solver]

---

### 51. Crowd and Multi-Agent Simulation

*Audience: graphics application developers.*

Navigation meshes (§70) compute flow fields telling each agent which direction to move. This section covers the per-agent algorithms that consume those fields and avoid collisions with other agents at run time.

### 51.1 ORCA: Optimal Reciprocal Collision Avoidance

ORCA (van den Berg et al. 2011) computes a collision-free velocity for each agent by solving a small 2D linear programme over velocity-obstacle half-planes from nearby neighbours:

```glsl
// orca.comp
layout(set=0,binding=0) readonly buffer Agents { AgentData a[]; };
layout(set=0,binding=1) writeonly buffer NewVel { vec2 nv[]; };
layout(local_size_x=64) in;

struct HalfPlane { vec2 point, normal; };

void main() {
    uint i=gl_GlobalInvocationID.x;
    vec2 pi=a[i].pos, vi=a[i].vel; float ri=a[i].radius;
    HalfPlane planes[MAX_NEIGHBOURS]; uint nPlanes=0;

    // iterate nearby agents via spatial hash (§27.2)
    ivec2 ci=ivec2(floor(pi/CELL_SIZE));
    for (int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell=hash2D(ci+ivec2(dx,dy));
        for (uint k=gridStart[cell];k<gridEnd[cell];k++) {
            uint j=gridAgents[k]; if(j==i) continue;
            vec2 relPos=a[j].pos-pi, relVel=vi-a[j].vel;
            float combR=ri+a[j].radius;
            // compute velocity obstacle boundary and closest-point offset u
            vec2 w=relVel-relPos/ORCA_TAU;
            float wlen=length(w);
            vec2 u=(combR/ORCA_TAU-wlen)*w/max(wlen,1e-6);
            planes[nPlanes++]=HalfPlane(vi+0.5*u, normalize(u));
            if(nPlanes>=MAX_NEIGHBOURS) break;
        }
    }
    nv[i]=solveLP2D(planes, nPlanes, a[i].prefVel, a[i].maxSpeed);
}
```

`solveLP2D` is an incremental 2D LP running entirely in registers — O(k) expected per agent. [Source: van den Berg et al. "Reciprocal n-Body Collision Avoidance", ISRR 2011; RVO2 https://gamma.cs.unc.edu/RVO2/]

### 51.2 Reynolds Boid Flocking

Three-rule boid steering (separation, alignment, cohesion) over a spatial-hash neighbourhood:

```glsl
// boids.comp
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    vec3 pi=pos[i], vi=vel[i];
    vec3 sep=vec3(0), ali=vec3(0), coh=vec3(0); uint n=0;
    // iterate neighbours within BOID_RADIUS via §27.2 grid
    for each neighbour j: {
        vec3 d=pi-pos[j];
        sep+=normalize(d)/max(length(d),0.01);
        ali+=vel[j]; coh+=pos[j]; n++;
    }
    if(n>0){ali=normalize(ali/float(n)); coh=normalize(coh/float(n)-pi);}
    vec3 steer=W_SEP*sep+W_ALI*ali+W_COH*coh+W_GOAL*normalize(goal-pi);
    vel[i]=normalize(vi+steer*DT)*BOID_SPEED;
    pos[i]=pi+vel[i]*DT;
}
```

[Source: Reynolds "Flocks, Herds, and Schools", SIGGRAPH 1987]

### 51.3 Agent Steering over Flow Fields

Blend ORCA collision-avoidance velocity with flow-field global direction:

```glsl
// steer_integrate.comp
layout(local_size_x=64) in;
void main() {
    uint i=gl_GlobalInvocationID.x;
    ivec2 ci=ivec2(floor(pos[i].xz/CELL_SIZE));
    vec2 flow=flowField[ci.y*GRID_W+ci.x].dir;   // §70.3
    vec2 orca=newVel[i];                            // §51.1
    vec2 steer=normalize(mix(flow,orca,ORCA_BLEND))*MAX_SPEED;
    vel[i].xz=steer;
    pos[i]+=vel[i]*DT;
}
```

### 51.4 GPU Crowd Rendering via Task Shaders

Each agent selects a pre-skinned LOD meshlet cluster; the task shader culls and emits only visible agents:

```glsl
// crowd_task.glsl
layout(local_size_x=32) in;
taskPayloadSharedEXT struct { uint agentIdx[32]; uint count; } payload;
void main() {
    uint i=gl_GlobalInvocationID.x;
    if(gl_LocalInvocationID.x==0) payload.count=0;
    barrier();
    AgentData ag=agents[i];
    bool visible=sphereInFrustum(ag.pos, AGENT_RADIUS)
              && length(ag.pos-cameraPos)<CULL_DIST;
    if(visible){ uint slot=atomicAdd(payload.count,1u); payload.agentIdx[slot]=i; }
    barrier();
    if(gl_LocalInvocationID.x==0) EmitMeshTasksEXT(payload.count,1,1);
}
```

Distant agents use billboard impostors (§10.7); close agents use full animated meshlets (§26.1–10.2). [Source: Reynolds 1987; van den Berg et al. 2011]

---

### 52. Fracture and Destruction

*Audience: graphics application developers.*

Fracture combines geometry processing (§3, §10, §49) with physics (§50) to produce visually plausible breaking and crumbling. GPU implementations pre-fracture offline for performance-critical paths and use runtime SDF crack propagation for interactive destruction.

### 52.1 Voronoi Pre-Fracture

Pre-computed Voronoi fracture assigns each voxel to its nearest fracture seed using 3D JFA (§3.9 extended to 3D). Each Voronoi cell becomes a convex fragment:

```glsl
// voronoi_fracture_3d.comp — 3D JFA step
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
layout(set=0,binding=0) buffer SeedMap { ivec4 seedMap[]; };  // xyz=nearest seed, w=unused
void main() {
    ivec3 vox=ivec3(gl_GlobalInvocationID);
    ivec4 best=seedMap[flatIdx(vox)]; float bestDist=1e30;
    for(int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        ivec3 n=clamp(vox+ivec3(dx,dy,dz)*JFA_STEP, ivec3(0), GRID_SIZE-1);
        ivec4 s=seedMap[flatIdx(n)];
        float d=length(vec3(vox-s.xyz));
        if(d<bestDist){bestDist=d; best=s;}
    }
    seedMap[flatIdx(vox)]=best;
}
```

After JFA, run GPU marching cubes (§3.2) on boundaries between adjacent seed regions to extract per-fragment surface meshes. Each fragment becomes an independent BLAS (§75.2) and rigid body (§50). [Source: Müller et al. "Real Time Dynamic Fracture with Volumetric Approximate Convex Decompositions", SIGGRAPH 2013]

### 52.2 Runtime Brittle Fracture via SDF Crack Propagation

Crack fronts advance as an SDF in a 3D grid, driven by a stress threshold:

```glsl
// crack_advance.comp
layout(set=0,binding=0) buffer CrackSDF { float sdf[]; };
layout(set=0,binding=1) readonly buffer Stress { float stress[]; };
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    uint v=flatIdx(ivec3(gl_GlobalInvocationID));
    if(sdf[v]>0.0 && sdf[v]<CRACK_SEARCH_RADIUS && stress[v]>FRACTURE_THRESHOLD)
        sdf[v]=-1.0;  // crack this voxel
}
```

Re-extract crack surface with GPU MC after each advance step; add new triangle geometry to the BLAS. [Source: Parker & O'Brien "Real-Time Deformation and Fracture in a Game Context", SCA 2009]

### 52.3 FEM Ductile Fracture

Per-element failure criterion on the corotational FEM from §49.2:

```glsl
// fem_fracture_check.comp
layout(set=0,binding=0) readonly buffer Stress { mat3 cauchyStress[]; };
layout(set=0,binding=1) buffer Failed { uint failedBits[]; };
layout(local_size_x=64) in;
void main() {
    uint t=gl_GlobalInvocationID.x;
    float maxEig=maxEigenvalue3x3sym(cauchyStress[t]);
    if(maxEig>FRACTURE_STRESS)
        atomicOr(failedBits[t/32], 1u<<(t%32));
}
```

Compact failed elements, patch crack surface (dual MC on failure boundary), add patch to BLAS. [Source: Irving et al. "Invertible Finite Elements for Robust Simulation", SCA 2004]

---

### 53. Rope and Cable Simulation

*Audience: graphics application developers.*

Ropes and cables are one-dimensional elastic bodies with stretch stiffness, bending stiffness, torsion, and friction/contact with other objects. Unlike hair (§42, many thin non-extensible strands) ropes are single extensible bodies with full Cosserat rod dynamics, catenary rest configuration, and pulley/constraint coupling.

### 53.1 Extensible Cosserat Rod Dynamics

Model the rope as N segments with position **d**ᵢ and material frame (bishop frame) **eᵢ¹, eᵢ², eᵢ³**. Forces: stretch (resists length change), bending (resists curvature change), torsion (resists frame twist):

```glsl
// rope_forces.comp — compute stretch and bending forces per segment
layout(set=0,binding=0) buffer Pos { vec4 pos[]; };   // w=invMass
layout(set=0,binding=1) buffer Frame { mat3 frame[]; };
layout(set=0,binding=2) writeonly buffer Force { vec3 force[]; };
layout(push_constant) uniform PC {
    float ks;   // stretch stiffness
    float kb;   // bending stiffness
    float restLen;
} pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i == 0u || i >= N_SEGS) return;

    // Stretch force
    vec3  e    = pos[i].xyz - pos[i-1].xyz;
    float len  = length(e);
    vec3  fStr = pc.ks * (len - pc.restLen) * normalize(e);
    force[i]   -= fStr;
    force[i-1] += fStr;

    // Bending force (discrete curvature via Bishop frame parallel transport)
    if (i < N_SEGS - 1) {
        vec3 t0 = normalize(pos[i  ].xyz - pos[i-1].xyz);
        vec3 t1 = normalize(pos[i+1].xyz - pos[i  ].xyz);
        vec3 kb_v = 2.0 * cross(t0, t1) / (1.0 + dot(t0, t1));
        force[i-1] += pc.kb * kb_v / len;
        force[i+1] -= pc.kb * kb_v / len;
    }
}
```

[Source: Bergou et al. "Discrete Elastic Rods", SIGGRAPH 2008; Kaufman et al. "Adaptive Nonlinearity for Collisions in Complex Rod Assemblies", SIGGRAPH 2014]

### 53.2 Catenary Initialisation

A rope under gravity with fixed endpoints hangs in a catenary. Initialise segment positions analytically to avoid transient oscillations from a straight-line start:

```glsl
// catenary_init.comp — compute catenary segment positions
layout(set=0,binding=0) writeonly buffer Pos { vec4 pos[]; };
layout(push_constant) uniform PC {
    vec3 p0, p1;    // endpoints
    float ropeLen;  float g;
    int   N;        // number of segments
} pc;
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    float t = float(i) / float(pc.N);
    // Solve catenary parameter a: L = a·sinh(d/(2a))·2 (binary search)
    float d  = length(pc.p1.xz - pc.p0.xz);
    float a  = solveCatenaryA(d, pc.ropeLen - abs(pc.p1.y - pc.p0.y));
    float x  = mix(pc.p0.x, pc.p1.x, t);
    float z  = mix(pc.p0.z, pc.p1.z, t);
    float xc = mix(-d*0.5, d*0.5, t);
    float y  = a * cosh(xc/a) + pc.p0.y - a;
    pos[i]   = vec4(x, y, z, i==0u||i==uint(pc.N)?0.0:1.0);  // w=invMass; 0=pinned
}
```

[Source: Irvine "Irvine on Cable Statics", 1981]

### 53.3 Pulley Constraint

A pulley maintains constant total rope length L_total = L_left + L_right while allowing the split point to slide:

```glsl
// pulley.comp — enforce pulley length constraint
layout(set=0,binding=0) buffer Pos { vec4 pos[]; };
layout(push_constant) uniform PC {
    uint leftEnd;     // index of rope endpoint on left side
    uint rightEnd;    // index of rope endpoint on right side
    uint pulleyVert;  // fixed pulley attachment point index
    float totalLen;
} pc;
layout(local_size_x=1) in;
void main() {
    float Ll = computeRopeLength(pc.leftEnd,  pc.pulleyVert);
    float Lr = computeRopeLength(pc.pulleyVert, pc.rightEnd);
    float err = (Ll + Lr) - pc.totalLen;
    // Distribute correction: shorten longer side, extend shorter side
    float frac = Ll / (Ll + Lr + 1e-5);
    applyLengthCorrection(pc.leftEnd,  pc.pulleyVert,  err * frac);
    applyLengthCorrection(pc.pulleyVert, pc.rightEnd, -err * (1.0-frac));
}
```

### 53.4 Rope-Rigid Body Contact

When the rope collides with a rigid body sphere (§50), apply position correction pushing the rope segment outside the sphere and a friction impulse tangent to the surface:

```glsl
// rope_contact.comp
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    for (uint b=0; b<N_BODIES; b++) {
        vec3  d    = pos[i].xyz - bodies[b].pos;
        float dist = length(d);
        float pen  = bodies[b].radius - dist;
        if (pen > 0.0 && pos[i].w > 0.0) {
            vec3 n    = normalize(d);
            pos[i].xyz += n * pen;
            // Friction: damp tangential velocity
            vec3 vt   = vel[i] - dot(vel[i],n)*n;
            vel[i]   -= vt * FRICTION;
        }
    }
}
```

[Source: Bergou et al. 2008; Spillmann & Teschner "An Adaptive Contact Model for the Robust Simulation of Knots", CGF 2008]

---

### 54. GPU Cloth Simulation

*Audience: graphics application developers.*

§39.6 introduced PBD cloth as a subsection of skeletal animation; §49.1 covered mass-spring systems. This section provides the dedicated treatment cloth deserves: woven fabric orthotropy, large-scale self-collision, runtime cutting and tearing, and aerodynamic forces. These distinguish cloth from both hair (§42, one-dimensional strands) and continuum FEM (§49, volumetric solids).

### 54.1 Orthotropic Cloth Constraints

Woven fabric resists stretch differently along warp (thread direction) and weft (cross-thread) axes. Encode per-face stiffness as an orthotropic tensor rather than a single Young's modulus:

```glsl
// cloth_stretch.comp — orthotropic stretch constraint per triangle
layout(set=0,binding=0) buffer Pos  { vec4 pos[];  };  // w = invMass
layout(set=0,binding=1) readonly buffer UV   { vec2 uv[];   };  // cloth-space coords
layout(set=0,binding=2) readonly buffer Tris { uvec3 tris[]; };
layout(push_constant) uniform PC { float ku; float kv; float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    uvec3 vi = tris[t];
    vec3  p0 = pos[vi.x].xyz, p1 = pos[vi.y].xyz, p2 = pos[vi.z].xyz;
    vec2  u0 = uv[vi.x],      u1 = uv[vi.y],       u2 = uv[vi.z];

    // Deformation gradient F = [p1-p0, p2-p0] * inv([u1-u0, u2-u0])
    mat2x3 dX = mat2x3(p1-p0, p2-p0);
    mat2   dU = mat2(u1-u0, u2-u0);
    mat2x3 F  = dX * inverse(dU);

    // Stretch along warp (u) and weft (v) axes
    float Cu = length(F[0]) - 1.0;  // stretch along warp
    float Cv = length(F[1]) - 1.0;  // stretch along weft
    vec3  nU = normalize(F[0]);
    vec3  nV = normalize(F[1]);

    // XPBD correction (simplified; full: account for constraint gradients)
    float corrU = pc.ku * Cu / (pos[vi.x].w + pos[vi.y].w + 1e-6);
    float corrV = pc.kv * Cv / (pos[vi.x].w + pos[vi.y].w + 1e-6);
    pos[vi.x].xyz -= corrU * nU * pos[vi.x].w;
    pos[vi.y].xyz += corrU * nU * pos[vi.y].w;
    pos[vi.x].xyz -= corrV * nV * pos[vi.x].w;
    pos[vi.z].xyz += corrV * nV * pos[vi.z].w;
}
```

[Source: Provot "Deformation Constraints in a Mass-Spring Model to Describe Rigid Cloth Behaviour", Graphics Interface 1995; Baraff & Witkin "Large Steps in Cloth Simulation", SIGGRAPH 1998]

### 54.2 Cloth Self-Collision via Spatial Hash

Self-collision at cloth densities (50k–500k triangles) uses the §27.2 spatial hash: each cloth vertex is hashed into a grid cell; pairs within one cell test for proximity:

```glsl
// cloth_selfcol.comp — vertex-vertex self-collision
layout(set=0,binding=0) buffer Pos { vec4 pos[]; };
layout(set=0,binding=1) readonly buffer GridStart { uint gStart[]; };
layout(set=0,binding=2) readonly buffer GridPts   { uint gPts[]; };
layout(push_constant) uniform PC { float thickness; float repulsion; } pc;
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    ivec3 ci = ivec3(floor(pos[i].xyz / pc.thickness));
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell = hashCell3(ci + ivec3(dx,dy,dz));
        for (uint k=gStart[cell]; k<gStart[cell+1]; k++) {
            uint j = gPts[k];
            if (j == i || sameTriangle(i,j)) continue;
            vec3  d = pos[i].xyz - pos[j].xyz;
            float dist = length(d);
            if (dist < pc.thickness && dist > 1e-5) {
                vec3 push = normalize(d) * (pc.thickness - dist) * pc.repulsion;
                if (pos[i].w > 0.0) pos[i].xyz += push * pos[i].w;
                if (pos[j].w > 0.0) pos[j].xyz -= push * pos[j].w;
            }
        }
    }
}
```

[Source: Bridson et al. "Robust Treatment of Collisions, Contact and Friction for Cloth Animation", SIGGRAPH 2002]

### 54.3 Aerodynamic Forces: Lift and Drag

Wind interaction applies lift and drag forces per cloth triangle based on the panel normal and relative wind velocity:

```glsl
// cloth_aero.comp — aerodynamic forces on cloth triangles
layout(set=0,binding=0) readonly buffer Pos  { vec4 pos[];  };
layout(set=0,binding=1) readonly buffer Vel  { vec3 vel[];  };
layout(set=0,binding=2) readonly buffer Tris { uvec3 t[];   };
layout(set=0,binding=3) buffer Force { vec3 force[]; };
layout(push_constant) uniform PC { vec3 wind; float rho; float cd; float cl; } pc;
layout(local_size_x=64) in;
void main() {
    uint ti  = gl_GlobalInvocationID.x;
    uvec3 vi = t[ti];
    vec3  p0 = pos[vi.x].xyz, p1 = pos[vi.y].xyz, p2 = pos[vi.z].xyz;
    vec3  vAvg = (vel[vi.x]+vel[vi.y]+vel[vi.z]) / 3.0;
    vec3  vRel = pc.wind - vAvg;
    vec3  n    = normalize(cross(p1-p0, p2-p0));
    float area = length(cross(p1-p0, p2-p0)) * 0.5;
    float vn   = dot(vRel, n);
    vec3  drag = 0.5 * pc.rho * pc.cd * area * length(vRel) * vRel;
    vec3  lift = 0.5 * pc.rho * pc.cl * area * vn * cross(cross(vRel,n), vRel);
    vec3  f    = (drag + lift) / 3.0;
    for (int k=0;k<3;k++) atomicAdd_vec3(force[vi[k]], f);
}
```

[Source: Leaf et al. "Interactive Design of Periodic Yarn-Level Cloth Patterns", SIGGRAPH 2018; Volino & Magnenat-Thalmann "Aerodynamic Effects on Cloth", Computers & Graphics 2001]

### 54.4 Runtime Cloth Cutting and Tearing

When edge stress exceeds a threshold, duplicate the shared vertex and update face connectivity:

```glsl
// cloth_tear.comp — detect overstressed edges, flag for topology update
layout(set=0,binding=0) readonly buffer Pos  { vec4 pos[]; };
layout(set=0,binding=1) readonly buffer Edges { uvec2 edges[]; };
layout(set=0,binding=2) buffer TearFlags { uint flags[]; };
layout(push_constant) uniform PC { float maxStress; float restLen; } pc;
layout(local_size_x=64) in;
void main() {
    uint e = gl_GlobalInvocationID.x;
    float stretch = length(pos[edges[e].x].xyz - pos[edges[e].y].xyz) / pc.restLen;
    if (stretch > pc.maxStress) atomicOr(flags[e/32], 1u << (e%32));
}
// CPU post-pass: for each flagged edge, duplicate vertex, retopologize adjacent faces,
// upload updated index buffer to GPU via vkCmdUpdateBuffer
```

[Source: Müller et al. "Real-Time Simulation of Deformation and Fracture of Stiff Materials", EG SCA 2001; Su et al. "Interactive Cutting of Deformable Objects in Virtual Environments", PRESENCE 2009]

---

### 55. GPU Fluid-Solid Coupling

*Audience: systems developers, graphics application developers.*

§48 (GPU fluid) and §50 (rigid body dynamics) operate independently. Two-way coupling means: (a) the solid moves fluid at the fluid-solid boundary (*no-slip* velocity boundary condition); (b) the fluid exerts pressure and viscous forces on the solid that accelerate it. This section covers the GPU geometry algorithms that implement this coupling.

### 55.1 Solid Boundary Voxelisation for Fluid Grids

At each frame, rasterise rigid body geometry into the fluid MAC grid cells. Voxels inside the solid are marked as solid cells; the fluid solver's pressure projection (§48.2) excludes them:

```glsl
// solid_vox_fluid.comp — mark MAC grid cells as solid or fluid
layout(set=0,binding=0) readonly buffer SolidTris { vec3 triV[]; };
layout(set=0,binding=1) buffer FluidMask  { uint solidMask[]; };  // 1 bit per cell
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; uint nTris; } pc;
layout(local_size_x=64) in;
void main() {
    uint t   = gl_GlobalInvocationID.x;
    if (t >= pc.nTris) return;
    vec3  v0 = triV[t*3], v1 = triV[t*3+1], v2 = triV[t*3+2];
    ivec3 bMin = ivec3(floor((min(v0,min(v1,v2))-pc.gridMin)/pc.dx));
    ivec3 bMax = ivec3(ceil( (max(v0,max(v1,v2))-pc.gridMin)/pc.dx));
    bMin = max(bMin, ivec3(0));  bMax = min(bMax, pc.dim-1);
    for (int z=bMin.z;z<=bMax.z;z++) for(int y=bMin.y;y<=bMax.y;y++) for(int x=bMin.x;x<=bMax.x;x++) {
        vec3 cen = pc.gridMin + (vec3(x,y,z)+0.5)*pc.dx;
        if (triBoxOverlap(cen, pc.dx*0.5, v0,v1,v2)) {
            uint flat = z*pc.dim.y*pc.dim.x + y*pc.dim.x + x;
            atomicOr(solidMask[flat/32], 1u << (flat%32));
        }
    }
}
```

### 55.2 No-Slip Velocity Boundary Condition

Fluid cells adjacent to a solid boundary inherit the solid's velocity at the boundary face. For face (i+½, j, k) between a fluid and solid cell:

```glsl
// noslip.comp — set boundary velocity in MAC grid from solid body velocity
layout(set=0,binding=0) buffer VelX { float vx[]; };   // MAC grid face velocities
layout(set=0,binding=1) readonly buffer SolidMask { uint smask[]; };
layout(set=0,binding=2) readonly buffer SolidVelGrid { vec3 sv[]; };  // interpolated solid vel
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 v   = ivec3(gl_GlobalInvocationID);
    uint  c   = flatIdx(v), cp1 = flatIdx(v+ivec3(1,0,0));
    bool  solidC  = bool((smask[c  /32]>>(c  %32))&1u);
    bool  solidN  = bool((smask[cp1/32]>>(cp1%32))&1u);
    if (solidC != solidN)  // face on solid-fluid boundary
        vx[flatFaceX(v)] = sv[solidC ? c : cp1].x;  // solid velocity at boundary
}
```

[Source: Bridson "Fluid Simulation for Computer Graphics", 2nd ed., §4; Batty et al. "A Fast Variational Framework for Accurate Solid-Fluid Coupling", SIGGRAPH 2007]

### 55.3 Fluid Pressure Force on Solid

Integrate fluid pressure over the solid surface to compute net force and torque on the rigid body. Sample pressure at each solid surface triangle's centroid from the MAC grid:

```glsl
// fluid_force.comp — accumulate pressure force on solid from fluid
layout(set=0,binding=0) readonly buffer Pressure { float p[]; };
layout(set=0,binding=1) readonly buffer SolidTris { vec3 tv[]; vec3 tn[]; };
layout(set=0,binding=2) buffer Force { vec3 force; vec3 torque; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; vec3 centroid; } pc;
layout(local_size_x=64) in;
void main() {
    uint t = gl_GlobalInvocationID.x;
    vec3 cen  = (tv[t*3]+tv[t*3+1]+tv[t*3+2])/3.0;
    float area = length(cross(tv[t*3+1]-tv[t*3],tv[t*3+2]-tv[t*3]))*0.5;
    vec3  n    = tn[t];
    ivec3 ci   = ivec3(floor((cen-pc.gridMin)/pc.dx));
    float pval = trilinearSamplePressure(p, ci, (cen-pc.gridMin)/pc.dx-vec3(ci));
    vec3  f    = -pval * n * area;
    atomicAdd_vec3(force,  f);
    atomicAdd_vec3(torque, cross(cen-pc.centroid, f));
}
```

### 55.4 Two-Way Coupling Loop

The per-frame GPU pipeline for two-way fluid-solid coupling:

```
1. vkCmdDispatch(solid_vox_fluid)    — §55.1: mark solid cells
2. vkCmdDispatch(noslip)             — §55.2: set boundary velocities
3. vkCmdDispatch(fluid_advect)       — §48.1: semi-Lagrangian advection
4. vkCmdDispatch(fluid_pressure_cg)  — §48.2: pressure projection (respects solid cells)
5. vkCmdDispatch(fluid_force)        — §55.3: compute force on solid
6. vkCmdDispatch(rigid_body_step)    — §50.3: integrate solid velocity with fluid force
7. vkCmdDispatch(blas_refit)         — §75.5: refit solid BLAS for next frame's ray queries
```

[Source: Batty et al. 2007; Robinson-Mosher et al. "Two-way Coupling of Fluids to Rigid and Deformable Solids and Shells", SIGGRAPH 2008]

---

### 56. Granular Material Simulation (DEM)

*Audience: systems developers, graphics application developers.*

The Discrete Element Method (DEM) simulates granular materials — sand, gravel, powder, pebbles — as collections of rigid spheres or polyhedral grains interacting via contact forces. Each particle-particle contact generates a spring-dashpot force; GPU parallelism comes from the same spatial hashing (§27.2) used in SPH (§48.4) and cloth self-collision (§54.2).

### 56.1 DEM Force Computation: Hertz Contact

For sphere-sphere contact, the Hertz contact model computes a non-linear spring force proportional to δ^(3/2) where δ is the overlap depth:

```glsl
// dem_force.comp — sphere-sphere Hertz contact forces
layout(set=0,binding=0) buffer Pos    { vec4 pos[];  };  // w = radius
layout(set=0,binding=1) buffer Vel    { vec3 vel[];  };
layout(set=0,binding=2) buffer AngVel { vec3 omega[]; };
layout(set=0,binding=3) buffer Force  { vec3 force[]; };
layout(set=0,binding=4) buffer Torque { vec3 torque[]; };
layout(set=0,binding=5) readonly buffer Pairs { uvec2 pairs[]; };
layout(push_constant) uniform PC { float E; float nu; float gamma; float mu_f; float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint p   = gl_GlobalInvocationID.x;
    uint i   = pairs[p].x, j = pairs[p].y;
    vec3  d  = pos[j].xyz - pos[i].xyz;
    float r  = pos[i].w + pos[j].w;
    float dl = length(d) - r;  // negative = overlap
    if (dl >= 0.0) return;
    float delta = -dl;
    vec3  n  = normalize(d);
    // Hertz normal force
    float Estar = pc.E / (2.0*(1.0-pc.nu*pc.nu));
    float Rstar = pos[i].w * pos[j].w / r;
    float kn    = 4.0/3.0 * Estar * sqrt(Rstar * delta);
    // Relative velocity at contact point
    vec3  vc = vel[j]-vel[i] + cross(omega[j],pos[j].w*(-n)) - cross(omega[i],pos[i].w*n);
    float vn = dot(vc, n);
    // Tangential (friction) force via Coulomb limit
    vec3  vt = vc - vn*n;
    vec3  Fn = (kn * delta + pc.gamma * vn) * n;
    vec3  Ft = clampLength(-pc.mu_f*length(Fn)*normalize(vt), length(Fn)*pc.mu_f);
    vec3  F  = Fn + Ft;
    atomicAdd_vec3(force[i], -F);
    atomicAdd_vec3(force[j],  F);
    atomicAdd_vec3(torque[i], cross(-pos[i].w*n, -F));
    atomicAdd_vec3(torque[j], cross( pos[j].w*n,  F));
}
```

[Source: Cundall & Strack "A Discrete Numerical Model for Granular Assemblies", Géotechnique 1979; Šmilauer et al. "Yade Documentation 2nd ed.", 2015]

### 56.2 Spatial Hashing for DEM Neighbour Search

Reuse the §27.2 / §54.2 spatial hash to find all particle pairs within contact range each frame. The hash cell size equals the maximum particle diameter:

```glsl
// dem_hash.comp — insert particles into spatial hash, find contacts
layout(set=0,binding=0) readonly buffer Pos   { vec4 pos[];  };  // w = radius
layout(set=0,binding=1) buffer HashTable { uint cells[HASH_SIZE]; };
layout(set=0,binding=2) buffer ContactList { uvec2 contacts[]; uint count; };
layout(push_constant) uniform PC { float cellSize; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    ivec3 ci = ivec3(floor(pos[i].xyz / pc.cellSize));
    // Check 3×3×3 neighbourhood for overlapping particles
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell = hashCell3(ci+ivec3(dx,dy,dz));
        for (uint k=hashStart[cell]; k<hashStart[cell+1]; k++) {
            uint j = hashPts[k];
            if (j <= i) continue;
            float d = length(pos[i].xyz-pos[j].xyz) - (pos[i].w+pos[j].w);
            if (d < 0.0) {
                uint slot = atomicAdd(count, 1u);
                contacts[slot] = uvec2(i,j);
            }
        }
    }
}
```

### 56.3 DEM–Fluid Coupling (CFD-DEM)

For wet granular flows (slurry, sediment transport), couple DEM with the fluid solver (§48 / §55): the fluid exerts drag on each particle; each particle displaces fluid from its volume:

```glsl
// dem_fluid_drag.comp — compute drag force on DEM sphere from local fluid velocity
layout(set=0,binding=0) readonly buffer Pos   { vec4 pos[];  };
layout(set=0,binding=1) readonly buffer VelP  { vec3 vp[];   };
layout(set=0,binding=2) uniform sampler3D fluidVel;  // MAC grid interpolated to particle pos
layout(set=0,binding=3) buffer Force { vec3 force[]; };
layout(push_constant) uniform PC { float rho; float nu; vec3 gridMin; float dx; } pc;
layout(local_size_x=64) in;
void main() {
    uint i    = gl_GlobalInvocationID.x;
    vec3  uvw = (pos[i].xyz - pc.gridMin) / (pc.dx * GRID_DIM);
    vec3  vf  = texture(fluidVel, uvw).xyz;
    vec3  vrel = vf - vp[i];
    float Re  = 2.0*pos[i].w*length(vrel)/pc.nu;
    float Cd  = (Re < 1000.0) ? 24.0/Re*(1.0+0.15*pow(Re,0.687)) : 0.44;
    float A   = 3.14159*pos[i].w*pos[i].w;
    vec3  Fd  = 0.5*pc.rho*Cd*A*length(vrel)*vrel;
    atomicAdd_vec3(force[i], Fd);
}
```

[Source: Di Felice "The Voidage Function for Fluid-Particle Interaction Systems", IJMF 1994; Kloss et al. "Models, Algorithms and Validation for Opensource DEM and CFD-DEM", Progress in CFD 2012]

### 56.4 Granular Surface Rendering: Instanced Sphere Splatting

Render millions of DEM spheres efficiently as GPU-instanced impostors: a single quad per particle, oriented toward the camera, with a ray-sphere intersection in the fragment shader:

```glsl
// dem_render.vert — instance sphere impostor billboard
layout(location=0) in vec2 quadUV;  // [-1,1]^2 screen quad
layout(location=0) flat out vec3 sphereCenter;
layout(location=1) flat out float sphereRadius;
layout(push_constant) uniform PC { mat4 viewProj; vec3 camPos; } pc;
void main() {
    uint i  = gl_InstanceIndex;
    sphereCenter = pos[i].xyz;
    sphereRadius = pos[i].w;
    // Billboard: offset quad in view space
    vec3 right = normalize(cross(pc.camPos-sphereCenter, vec3(0,1,0)));
    vec3 up    = cross(normalize(pc.camPos-sphereCenter), right);
    vec3 wp    = sphereCenter + (right*quadUV.x + up*quadUV.y)*sphereRadius*1.01;
    gl_Position = pc.viewProj * vec4(wp,1.0);
}
// Fragment: ray-sphere intersection → depth override for correct occlusion
```

[Source: Gumhold "Splatting Illuminated Ellipsoids with Depth Correction", VMV 2003; Müller "Particle-Based Fluid Simulation for Interactive Applications", SCA 2003]

---

### 57. Terrain Sculpting and Hydraulic Erosion

*Audience: graphics application developers.*

Procedural terrain generation (§71 covers runtime rendering; §73 covers ocean simulation) leaves open the sculpting and erosion simulation that shapes the terrain heightfield before rendering. GPU hydraulic erosion simulates millions of water droplets carrying sediment down gradient-following paths; thermal erosion redistributes material based on angle-of-repose; and interactive sculpting applies brush strokes at render resolution.

### 57.1 Particle-Based Hydraulic Erosion

Each virtual water droplet carries sediment, picks up material where excess capacity exists, and deposits where at capacity. Millions of droplets run in parallel, each as an independent compute thread:

```glsl
// erosion_droplet.comp — one GPU hydraulic erosion droplet simulation
layout(set=0,binding=0) buffer Heightmap { float h[]; };
layout(set=0,binding=1) buffer Sediment  { float sed[]; };
layout(set=0,binding=2) readonly buffer StartPos { vec2 startPos[]; };
layout(push_constant) uniform PC {
    int W; int H; float inertia; float capacity;
    float deposition; float erosion; float evaporation; float gravity;
    int maxSteps;
} pc;
layout(local_size_x=64) in;
void main() {
    uint di = gl_GlobalInvocationID.x;
    vec2 pos = startPos[di];
    vec2 dir = vec2(0.0);
    float water=1.0, speed=0.0, sediment=0.0;
    for (int s=0; s<pc.maxSteps && water>0.01; s++) {
        ivec2 c = ivec2(pos);
        if (c.x<1||c.y<1||c.x>=pc.W-1||c.y>=pc.H-1) break;
        // Bilinear gradient of heightmap
        vec2 grad = vec2(
            h[(c.y  )*pc.W+(c.x+1)] - h[(c.y  )*pc.W+(c.x-1)],
            h[(c.y+1)*pc.W+(c.x  )] - h[(c.y-1)*pc.W+(c.x  )]) * 0.5;
        dir = normalize(mix(-grad, dir, pc.inertia));
        vec2 npos = pos + dir;
        float dh  = bilinearH(h, npos, pc.W) - bilinearH(h, pos, pc.W);
        float cap = max(-dh, 0.01) * speed * water * pc.capacity;
        if (sediment > cap || dh > 0.0) {
            float dep = (dh > 0.0) ? min(dh, sediment) : (sediment-cap)*pc.deposition;
            atomicAdd_float(h[c.y*pc.W+c.x], dep);
            sediment -= dep;
        } else {
            float ero = min((cap-sediment)*pc.erosion, -dh);
            atomicAdd_float(h[c.y*pc.W+c.x], -ero);
            sediment += ero;
        }
        speed  = sqrt(max(speed*speed + dh*pc.gravity, 0.0));
        water *= (1.0 - pc.evaporation);
        pos    = npos;
    }
}
```

[Source: Olsen "Realtime Procedural Terrain Generation", 2004; Beneš & Forsbach "Layered Data Representation for Visual Simulation of Terrain Erosion", SCCG 2001; GPU erosion: Št'ava et al. "Interactive Terrain Modeling Using Hydraulic Erosion", SCA 2008]

### 57.2 Thermal Erosion

Thermal erosion redistributes material from cells whose slope exceeds the angle of repose (typically 30–40° for soil). Each grid cell checks its 4 neighbours:

```glsl
// thermal_erosion.comp — redistribute material above angle of repose
layout(set=0,binding=0) buffer Heightmap { float h[]; };
layout(push_constant) uniform PC { int W; int H; float dx; float tanAngle; float rate; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    if (id.x<=0||id.y<=0||id.x>=pc.W-1||id.y>=pc.H-1) return;
    uint  c   = id.y*pc.W+id.x;
    float hc  = h[c];
    float dTotal = 0.0; float excess = 0.0;
    const ivec2 DIRS[4] = {ivec2(1,0),ivec2(-1,0),ivec2(0,1),ivec2(0,-1)};
    float dh[4];
    for (int d=0;d<4;d++) {
        uint nb = (id.y+DIRS[d].y)*pc.W+(id.x+DIRS[d].x);
        dh[d]  = hc - h[nb];
        if (dh[d] > pc.tanAngle*pc.dx) { dTotal+=dh[d]; excess+=dh[d]-pc.tanAngle*pc.dx; }
    }
    if (dTotal < 1e-6) return;
    float move = excess * pc.rate * 0.5;
    atomicAdd_float(h[c], -move);
    for (int d=0;d<4;d++) if (dh[d] > pc.tanAngle*pc.dx) {
        uint nb = (id.y+DIRS[d].y)*pc.W+(id.x+DIRS[d].x);
        atomicAdd_float(h[nb], move * dh[d]/dTotal);
    }
}
```

[Source: Musgrave et al. "The Synthesis and Rendering of Eroded Fractal Terrains", SIGGRAPH 1989; Št'ava et al. 2008]

### 57.3 Interactive Terrain Sculpting Brushes

Real-time sculpting applies a Gaussian-weighted height offset inside a brush radius. The brush follows the user's cursor position at interactive rates:

```glsl
// sculpt_brush.comp — apply sculpt brush to heightmap
layout(set=0,binding=0) buffer Heightmap { float h[]; };
layout(push_constant) uniform PC {
    vec2  centre;     // brush centre in heightmap coordinates
    float radius;     // brush radius in pixels
    float strength;   // height change per frame
    float falloff;    // Gaussian sigma / radius ratio
    int   W; int H;
    int   mode;       // 0=raise 1=lower 2=smooth 3=flatten
} pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  d   = vec2(id) - pc.centre;
    float r   = length(d) / pc.radius;
    if (r > 1.0) return;
    float w   = exp(-r*r/(2.0*pc.falloff*pc.falloff));
    uint  flat= id.y*pc.W+id.x;
    if      (pc.mode==0) h[flat] += w * pc.strength;
    else if (pc.mode==1) h[flat] -= w * pc.strength;
    else if (pc.mode==2) {  // smooth: Laplacian step
        float avg = (h[flat-1]+h[flat+1]+h[flat-pc.W]+h[flat+pc.W])*0.25;
        h[flat]   = mix(h[flat], avg, w * pc.strength);
    }
    else if (pc.mode==3) h[flat] = mix(h[flat], pc.strength, w);  // flatten to target height
}
```

[Source: Gain et al. "Terrain Sketching", I3D 2009; Hnaidi et al. "Feature Based Terrain Generation Using Diffusion Equation", CGF 2010]

### 57.4 Heightmap Normal and LOD Update

After sculpting or erosion, recompute normals and terrain LOD tiles (§71 pattern) for the dirty region:

```glsl
// terrain_normals.comp — recompute normals for a dirty tile after sculpting
layout(set=0,binding=0) readonly buffer Heightmap { float h[]; };
layout(set=0,binding=1) writeonly buffer Normals { vec3 nor[]; };
layout(push_constant) uniform PC { int W; float dx; ivec4 dirtyRect; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy) + pc.dirtyRect.xy;
    if (any(greaterThanEqual(id, pc.dirtyRect.zw))) return;
    uint  c   = id.y*pc.W+id.x;
    float hL  = h[c-1], hR=h[c+1], hD=h[c-pc.W], hU=h[c+pc.W];
    vec3  n   = normalize(vec3(hL-hR, 2.0*pc.dx, hD-hU));
    nor[c]    = n;
}
// vkCmdUpdateBuffer the dirty tile heightmap → GPU; then dispatch normal update
// Optionally: rebuild LOD quadtree for the dirty tile (§71 clipmap update)
```

[Source: Losasso & Hoppe "Geometry Clipmaps: Terrain Rendering Using Nested Regular Grids", SIGGRAPH 2004; Bruneton & Neyret "Real-time Realistic Ocean Lighting using Seamless Transitions from Geometry to BRDF", EG 2010]

---

### 58. GPU Fluid Surface Extraction

*Audience: graphics application developers.*

§48 (SPH/FLIP fluid simulation) produces particle positions and velocities; §65 (isosurface extraction) expects a grid scalar field. Bridging them for liquid rendering requires: (1) splatting anisotropic kernels from particles into a density grid; (2) running marching cubes on that grid; (3) smoothing the resulting mesh. This pipeline differs from generic isosurface extraction because the kernels are oriented by the local velocity gradient to produce smooth water surfaces without blobby artefacts.

### 58.1 Isotropic Density Splatting

The simplest approach: for each particle i, add a radial kernel contribution to nearby grid cells. Parallelise over particles with atomic accumulation:

```glsl
// fluid_splat_iso.comp — isotropic kernel splatting into density grid
layout(set=0,binding=0) readonly buffer Particles { vec4 pos[]; };  // w = mass
layout(set=0,binding=1) buffer Density { float rho[]; };
layout(push_constant) uniform PC {
    vec3 gridMin; float dx; ivec3 dim; float h; uint N;
} pc;
layout(local_size_x=64) in;
void main() {
    uint p   = gl_GlobalInvocationID.x;
    if (p >= pc.N) return;
    vec3  pi  = pos[p].xyz;
    float m   = pos[p].w;
    ivec3 ci  = ivec3(floor((pi - pc.gridMin) / pc.dx));
    int   r   = int(ceil(pc.h / pc.dx));
    for (int dz=-r;dz<=r;dz++) for(int dy=-r;dy<=r;dy++) for(int dx=-r;dx<=r;dx++) {
        ivec3 nb = ci + ivec3(dx,dy,dz);
        if (any(lessThan(nb,ivec3(0)))||any(greaterThanEqual(nb,pc.dim))) continue;
        vec3  xg = pc.gridMin + (vec3(nb)+0.5)*pc.dx;
        float r2 = dot(pi-xg,pi-xg);
        if (r2 > pc.h*pc.h) continue;
        float q  = sqrt(r2)/pc.h;
        float W  = max(0.0, 1.0-q*q) * (315.0/(64.0*3.14159*pow(pc.h,3.0)));
        atomicAdd_float(rho[nb.z*pc.dim.y*pc.dim.x + nb.y*pc.dim.x + nb.x], m*W);
    }
}
```

[Source: Zhu & Bridson "Animating Sand as a Fluid", SIGGRAPH 2005; Müller et al. "Particle-Based Fluid Simulation", SCA 2003]

### 58.2 Anisotropic Kernel Splatting

Yu & Turk (2013) orient each particle's kernel using the local velocity gradient covariance matrix, producing smooth surface sheets rather than blobs:

```glsl
// fluid_splat_aniso.comp — anisotropic kernel splatting (Yu & Turk 2013)
layout(set=0,binding=0) readonly buffer Particles { vec3 pos[]; };
layout(set=0,binding=1) readonly buffer KernelG   { mat3 G[];   };  // per-particle anisotropy
layout(set=0,binding=2) buffer Density { float rho[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    if (p >= pc.N) return;
    mat3  Gp = G[p];
    // Bounding box of anisotropic kernel in grid space
    vec3  ext = vec3(length(Gp[0]), length(Gp[1]), length(Gp[2]));
    ivec3 ci  = ivec3(floor((pos[p]-pc.gridMin)/pc.dx));
    ivec3 r   = ivec3(ceil(ext/pc.dx)) + 1;
    for (int dz=-r.z;dz<=r.z;dz++) for(int dy=-r.y;dy<=r.y;dy++) for(int dx=-r.x;dx<=r.x;dx++) {
        ivec3 nb  = ci + ivec3(dx,dy,dz);
        if (any(lessThan(nb,ivec3(0)))||any(greaterThanEqual(nb,pc.dim))) continue;
        vec3  xg  = pc.gridMin + (vec3(nb)+0.5)*pc.dx;
        vec3  d   = Gp * (xg - pos[p]);  // transform to kernel space
        float r2  = dot(d,d);
        if (r2 >= 1.0) continue;
        float W   = pow(1.0-r2, 3.0) * (315.0/(64.0*3.14159));
        atomicAdd_float(rho[flatIdx(nb,pc.dim)], W * determinant(Gp));
    }
}
```

[Source: Yu & Turk "Reconstructing Surfaces of Particle-Based Fluids Using Anisotropic Kernels", ACM TOG 2013]

### 58.3 Screen-Space Fluid Rendering (Depth Smoothing)

An alternative to volumetric splatting: render particle spheres to a depth buffer, blur the depth field with a curvature-flow filter, then shade the resulting surface as a thin dielectric:

```glsl
// fluid_depthsmooth.comp — bilateral curvature-flow filter on fluid depth buffer
layout(set=0,binding=0) uniform sampler2D fluidDepth;   // particle sphere depth
layout(set=0,binding=1, r32f) uniform writeonly image2D smoothedDepth;
layout(push_constant) uniform PC { float sigmaD; float sigmaZ; int radius; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float zc   = texelFetch(fluidDepth, pix, 0).r;
    if (zc <= 0.0) { imageStore(smoothedDepth, pix, vec4(0)); return; }
    float sum  = 0.0, wSum = 0.0;
    for (int dy=-pc.radius;dy<=pc.radius;dy++) for(int dx=-pc.radius;dx<=pc.radius;dx++) {
        float zn  = texelFetch(fluidDepth, pix+ivec2(dx,dy), 0).r;
        if (zn <= 0.0) continue;
        float wD  = exp(-(dx*dx+dy*dy)/(2.0*pc.sigmaD*pc.sigmaD));
        float wZ  = exp(-(zc-zn)*(zc-zn)/(2.0*pc.sigmaZ*pc.sigmaZ));
        sum  += wD*wZ*zn; wSum += wD*wZ;
    }
    imageStore(smoothedDepth, pix, vec4(wSum>0.0 ? sum/wSum : zc));
}
```

[Source: van der Laan et al. "Screen Space Fluid Rendering with Curvature Flow", I3D 2009; Green "Screen Space Fluid Rendering", GDC 2010]

### 58.4 Surface Normal and Shading

Reconstruct the surface normal from the smoothed depth buffer, then shade as a thin glass/water surface using Fresnel reflectance and refraction with a cubemap:

```glsl
// fluid_shade.frag — shade fluid surface from smoothed depth
uniform sampler2D smoothDepth;
uniform samplerCube envMap;
uniform mat4 invProj;
in vec2 texUV;
void main() {
    float z  = texture(smoothDepth, texUV).r;
    if (z <= 0.0) discard;
    // Reconstruct view-space position
    vec4  vp = invProj * vec4(texUV*2-1, z*2-1, 1); vp /= vp.w;
    // Finite-difference normal from depth buffer
    vec3  n  = normalize(vec3(
        dFdx(vp.z), dFdy(vp.z), -length(vec2(dFdx(vp.z),dFdy(vp.z)))));
    // Fresnel reflection + refraction
    vec3  v  = normalize(-vp.xyz);
    float F  = 0.02 + 0.98*pow(1.0-max(dot(n,v),0.0), 5.0);
    vec3  r  = reflect(-v, n);
    vec3  t  = refract(-v, n, 1.0/1.33);  // water IOR
    fragColor = vec4(mix(texture(envMap,t).rgb, texture(envMap,r).rgb, F), 1.0);
}
```

[Source: Green 2010; James & Fatahalian "Precomputing Interactive Dynamic Deformable Scenes", SIGGRAPH 2003]

---

### 59. Continuous Collision Detection (CCD)

*Audience: systems developers, graphics application developers.*

Discrete collision detection (§64) checks for overlap at a single instant; fast-moving or thin objects may tunnel through each other between frames. Continuous collision detection finds the time of first contact (TOC) τ ∈ [0,1] within a timestep by solving the polynomial distance equation between swept primitives. GPU parallelism runs independent TOC solves for all candidate pairs simultaneously.

### 59.1 Vertex–Triangle CCD via Conservative Advancement

Conservative advancement (Zhang et al. 2007) iteratively advances the time along the minimum separation distance until contact or the interval is exhausted:

```glsl
// ccd_vertex_tri.comp — vertex-triangle CCD via conservative advancement
layout(set=0,binding=0) readonly buffer PosT0 { vec3 p0[]; };   // positions at t=0
layout(set=0,binding=1) readonly buffer PosT1 { vec3 p1[]; };   // positions at t=1
layout(set=0,binding=2) readonly buffer Pairs  { uvec4 vt[];    };  // vertex idx + 3 tri verts
layout(set=0,binding=3) writeonly buffer TOC   { float toc[];   };  // time of contact (-1=none)
layout(local_size_x=64) in;
void main() {
    uint pi = gl_GlobalInvocationID.x;
    uint vi = vt[pi].x;
    uint ai = vt[pi].y, bi = vt[pi].z, ci = vt[pi].w;
    float t  = 0.0;
    for (int iter=0; iter<64; iter++) {
        vec3 v  = mix(p0[vi],p1[vi],t);
        vec3 a  = mix(p0[ai],p1[ai],t);
        vec3 b  = mix(p0[bi],p1[bi],t);
        vec3 c  = mix(p0[ci],p1[ci],t);
        float d = pointTriDist(v, a, b, c);
        if (d < 1e-6) { toc[pi]=t; return; }
        // Motion bound: max speed × remaining time
        float vMax = max(length(p1[vi]-p0[vi]),
                     max(length(p1[ai]-p0[ai]),
                     max(length(p1[bi]-p0[bi]), length(p1[ci]-p0[ci]))));
        float dt   = d / (vMax + 1e-8);
        t += dt;
        if (t >= 1.0) break;
    }
    toc[pi] = -1.0;  // no contact
}
```

[Source: Zhang et al. "Continuous Collision Detection for Rigid and Articulated Bodies", SIGGRAPH 2007; Tang et al. "ICCD: Interactive Continuous Collision Detection between Deformable Models Using Connectivity-Based Culling", IEEE TVCG 2009]

### 59.2 Edge–Edge CCD via Cubic Root Finding

For edge-edge contact between two moving line segments, the coplanarity condition produces a cubic polynomial in τ whose roots are the candidate contact times:

```glsl
// ccd_edge_edge.comp — edge-edge CCD via cubic root isolation
layout(set=0,binding=0) readonly buffer PosT0 { vec3 p0[]; };
layout(set=0,binding=1) readonly buffer PosT1 { vec3 p1[]; };
layout(set=0,binding=2) readonly buffer EEPairs { uvec4 ee[]; };
layout(set=0,binding=3) writeonly buffer TOC { float toc[]; };
layout(local_size_x=64) in;

// Cubic coefficients of (e1(τ) × e2(τ)) · (v1(τ) - v3(τ)) = 0
void cubicCoeffs(vec3 a0,vec3 a1,vec3 b0,vec3 b1,vec3 c0,vec3 c1,vec3 d0,vec3 d1,
                 out float A, out float B, out float C, out float D) {
    vec3 da=a1-a0, db=b1-b0, dc=c1-c0, dd=d1-d0;
    // A*t³ + B*t² + C*t + D = scalar triple product of swept edge vectors
    vec3 e1=b0-a0, e2=d0-c0, r=c0-a0;
    A = dot(cross(da-db, dc-dd), da-db) + dot(cross(e1,dc-dd), da-db)
      + dot(cross(da-db, e2), da-db);
    // (simplified — full derivation in Bridson et al. 2002)
    B = dot(cross(e1,e2), da-db) + dot(cross(da-db,e2),r)
      + dot(cross(e1,dc-dd),r);
    C = dot(cross(e1,e2),r) + dot(cross(da-db,e2),r);
    D = dot(cross(e1,e2),r);
}
void main() {
    uint pi = gl_GlobalInvocationID.x;
    float A,B,C,D;
    cubicCoeffs(p0[ee[pi].x],p1[ee[pi].x], p0[ee[pi].y],p1[ee[pi].y],
                p0[ee[pi].z],p1[ee[pi].z], p0[ee[pi].w],p1[ee[pi].w], A,B,C,D);
    float roots[3]; int nr = solveCubic(A,B,C,D,roots);
    float earliest = 1.0;
    for (int r=0; r<nr; r++) if (roots[r]>0&&roots[r]<earliest&&verifyContact(roots[r],pi)) earliest=roots[r];
    toc[pi] = (earliest < 1.0) ? earliest : -1.0;
}
```

[Source: Bridson et al. "Robust Treatment of Collisions, Contact and Friction for Cloth Animation", SIGGRAPH 2002; Provot "Collision and Self-Collision Handling in Cloth Model Dedicated to Design Garments", CAS 1997]

### 59.3 Broadphase CCD via Swept AABB

Before running exact CCD, cull pairs whose swept bounding boxes (union of AABB at t=0 and t=1) do not overlap:

```glsl
// ccd_broadphase.comp — swept AABB overlap test for CCD culling
layout(set=0,binding=0) readonly buffer AABB_T0 { vec6 aabb0[]; };
layout(set=0,binding=1) readonly buffer AABB_T1 { vec6 aabb1[]; };
layout(set=0,binding=2) readonly buffer Candidates { uvec2 pairs[]; };
layout(set=0,binding=3) buffer Survivors { uvec2 surv[]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint pi  = gl_GlobalInvocationID.x;
    uint a   = pairs[pi].x, b = pairs[pi].y;
    // Swept AABB: union of t=0 and t=1 box
    vec3 aminS = min(aabb0[a].xyz, aabb1[a].xyz);
    vec3 amaxS = max(aabb0[a].xyz+aabb0[a].xyz, aabb1[a].xyz+aabb1[a].xyz);
    vec3 bminS = min(aabb0[b].xyz, aabb1[b].xyz);
    vec3 bmaxS = max(aabb0[b].xyz+aabb0[b].xyz, aabb1[b].xyz+aabb1[b].xyz);
    if (all(lessThan(aminS,bmaxS)) && all(lessThan(bminS,amaxS))) {
        uint slot = atomicAdd(count, 1u);
        surv[slot] = pairs[pi];
    }
}
```

[Source: Ericson "Real-Time Collision Detection", Morgan Kaufmann 2005, §5.3]

### 59.4 CCD Response: Impact Zone Projection

After finding τ, project the impacting vertices out of collision along the contact normal and apply impulses. The impact zone (Bridson et al. 2002) groups all vertices involved in a contact cluster:

```glsl
// ccd_response.comp — apply collision response at TOC τ
layout(set=0,binding=0) buffer Pos { vec3 pos[]; };
layout(set=0,binding=1) buffer Vel { vec3 vel[]; };
layout(set=0,binding=2) readonly buffer Contacts { CCDContact contacts[]; };
layout(push_constant) uniform PC { float restitution; float friction; } pc;
layout(local_size_x=64) in;
void main() {
    uint c  = gl_GlobalInvocationID.x;
    CCDContact ct = contacts[c];
    vec3  n = ct.normal;
    float vRel = dot(vel[ct.v] - vel[ct.a]*ct.w0 - vel[ct.b]*ct.w1 - vel[ct.c]*ct.w2, n);
    if (vRel >= 0.0) return;  // separating: no impulse needed
    float j = -(1.0+pc.restitution)*vRel /
              (ct.invMassV + ct.w0*ct.w0*ct.invMassA + ct.w1*ct.w1*ct.invMassB + ct.w2*ct.w2*ct.invMassC);
    atomicAdd_vec3(vel[ct.v],  j*n*ct.invMassV);
    atomicAdd_vec3(vel[ct.a], -j*n*ct.invMassA*ct.w0);
    atomicAdd_vec3(vel[ct.b], -j*n*ct.invMassB*ct.w1);
    atomicAdd_vec3(vel[ct.c], -j*n*ct.invMassC*ct.w2);
}
```

[Source: Bridson et al. 2002; Harmon et al. "Robust Treatment of Simultaneous Collisions", SIGGRAPH 2008]

---

### 60. Minkowski Sum and GPU Motion Planning

*Audience: systems developers.*

The Minkowski sum A⊕B of two convex shapes A and B is the shape swept by placing B's centre on every point of A; it is the configuration-space obstacle (C-obstacle) for robot motion planning. GPU computation of Minkowski sums on convex polytopes and GPU-parallel probabilistic roadmap (PRM) and RRT planning enable real-time collision-free path queries.

### 60.1 Minkowski Sum of Convex Polytopes (GJK-Based)

For two convex polytopes A and B with n_A and n_B vertices respectively, the Minkowski sum has at most O(n_A · n_B) vertices. GPU enumerates candidate extreme points and culls non-hull points:

```glsl
// mink_sum_vertices.comp — generate Minkowski sum vertex candidates
layout(set=0,binding=0) readonly buffer VertsA { vec3 vA[]; };
layout(set=0,binding=1) readonly buffer VertsB { vec3 vB[]; };
layout(set=0,binding=2) writeonly buffer SumVerts { vec3 sv[]; };
layout(local_size_x=16,local_size_y=16) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint j  = gl_GlobalInvocationID.y;
    if (i >= N_A || j >= N_B) return;
    sv[i*N_B+j] = vA[i] + vB[j];  // Minkowski sum vertex candidate
}
// Then run §106 GPU convex hull on sv[] to get the actual Minkowski sum hull
```

[Source: Edelsbrunner "Algorithms in Combinatorial Geometry", Springer 1987; van den Bergen "Collision Detection in Interactive 3D Environments", Morgan Kaufmann 2004]

### 60.2 Minkowski Difference for GJK Collision

The GJK distance algorithm (§24) operates implicitly on the Minkowski difference A⊖B = A⊕(−B) via support functions. GPU parallelises GJK for a large batch of shape pairs simultaneously:

```glsl
// gjk_batch.comp — GJK distance for a batch of (convex A, convex B) pairs
layout(set=0,binding=0) readonly buffer ShapeA { ConvexShape shapesA[]; };
layout(set=0,binding=1) readonly buffer ShapeB { ConvexShape shapesB[]; };
layout(set=0,binding=2) writeonly buffer Dist  { float dist[]; };  // negative = penetrating
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    ConvexShape A = shapesA[p], B = shapesB[p];
    vec3 v  = A.centre - B.centre;
    vec3 simplex[4]; int nSimplex = 0;
    for (int iter=0; iter<64; iter++) {
        // Support on Minkowski difference: sup_{A⊖B}(v) = sup_A(v) - sup_B(-v)
        vec3 w  = support(A, v) - support(B, -v);
        if (dot(v,v) - dot(w,v) < 1e-8) { dist[p]=length(v); return; }
        addToSimplex(simplex, nSimplex, w);
        v = nearestSimplexPoint(simplex, nSimplex);
    }
    dist[p] = -1.0;  // penetrating
}
```

[Source: Gilbert et al. "A Fast Procedure for Computing the Distance Between Complex Objects in Three-Dimensional Space", IEEE T-RA 1988; §8]

### 60.3 Probabilistic Roadmap (PRM) on GPU

GPU-parallel PRM builds a configuration-space roadmap by sampling random configurations, testing each for collision (§60.2 GJK batch), and connecting collision-free neighbours:

```glsl
// prm_sample.comp — sample random configurations and test collision-free
layout(set=0,binding=0) readonly buffer BlueNoise { vec2 bn[]; };
layout(set=0,binding=1) readonly buffer Obstacles { ConvexShape obs[]; };
layout(set=0,binding=2) buffer Roadmap { vec3 configs[]; uint count; };
layout(push_constant) uniform PC { vec3 cMin; vec3 cRange; uint N_SAMPLE; uint N_OBS; } pc;
layout(local_size_x=64) in;
void main() {
    uint s   = gl_GlobalInvocationID.x;
    if (s >= pc.N_SAMPLE) return;
    vec3 q   = pc.cMin + vec3(bn[s*3%BN_SIZE],bn[(s*3+1)%BN_SIZE].x,bn[(s*3+2)%BN_SIZE].x)*pc.cRange;
    // Test q against all obstacles via GJK
    bool free = true;
    for (uint o=0; o<pc.N_OBS && free; o++) {
        ConvexShape robot = robotAt(q);  // robot shape at config q
        if (gjkPenetrating(robot, obs[o])) free=false;
    }
    if (free) { uint slot=atomicAdd(count,1u); configs[slot]=q; }
}
```

[Source: Kavraki et al. "Probabilistic Roadmaps for Path Planning in High-Dimensional Configuration Spaces", IEEE T-RA 1996; Lauterbach et al. "gProximity: Hierarchical GPU-Based Operations for Collision and Distance Queries", EG 2010]

### 60.4 RRT Nearest-Neighbour on GPU

Rapidly-exploring random tree (RRT) nearest-neighbour search — finding the closest existing tree node to a random sample — is the bottleneck. GPU parallelises the linear search across all tree nodes:

```glsl
// rrt_nearest.comp — find nearest tree node to random sample q_rand
layout(set=0,binding=0) readonly buffer Tree   { vec3 nodes[]; uint treeSize; };
layout(set=0,binding=1) readonly buffer QRand  { vec3 qRand;  };
layout(set=0,binding=2) buffer NearestIdx { uint idx; float dist; };
layout(local_size_x=256) in;
shared uint sIdx[256]; shared float sDist[256];
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint li = gl_LocalInvocationID.x;
    float d = (i < treeSize) ? length(nodes[i]-qRand) : 1e30;
    sIdx[li]=i; sDist[li]=d; barrier();
    for (uint s=128u;s>0u;s>>=1) {
        if (li<s && sDist[li+s]<sDist[li]) { sDist[li]=sDist[li+s]; sIdx[li]=sIdx[li+s]; }
        barrier();
    }
    if (li==0) { atomicMin_float_idx(dist, idx, sDist[0], sIdx[0]); }
}
```

[Source: LaValle "Rapidly-Exploring Random Trees: A New Tool for Path Planning", TR 1998; GPU RRT: Murray et al. "GPU Acceleration of the Fast Marching Method", Parallel Computing 2009]

---


---

## VII. SDF and Volumetric Methods

Signed Distance Functions provide a unified representation for implicit geometry, collision proximity queries, and volumetric rendering effects. This category covers level-set evolution equations for simulating moving interfaces (flame fronts, water surfaces), TSDF volumetric reconstruction from depth sensor streams, isosurface extraction algorithms (Marching Cubes, Dual Contouring) that recover triangle meshes from scalar fields, SDF-based proximity collision detection, volumetric ray marching for clouds and participating media, and GPU ambient occlusion estimated by ray marching against an SDF volume. The GPU sparse narrow-band level-set solver is the real-time counterpart to OpenVDB-style offline VFX pipelines.

### 61. Level Set Methods

*Audience: systems developers, graphics application developers.*

Level set methods (Osher & Sethian 1988) represent surfaces implicitly as the zero crossing of a scalar field φ(x,t) stored on a grid. Unlike JFA (§3.4), which constructs SDFs from point sources, LSM *evolves* the SDF forward in time using a PDE. Key applications: fluid free surfaces (§48 uses MAC grid + LSM hybrid), phase-field fracture (§52), topology-changing surface animation.

### 61.1 Level Set Advection

The fundamental LSM PDE for a velocity field **v**:

∂φ/∂t + **v** · ∇φ = 0

Discretise using semi-Lagrangian advection (same stencil as §48.1 MAC advection, but applied to the scalar SDF field):

```glsl
// ls_advect.comp — semi-Lagrangian level set advection
layout(set=0,binding=0) uniform sampler3D phiIn;
layout(set=0,binding=1) uniform sampler3D velField;
layout(set=0,binding=2, r32f) uniform writeonly image3D phiOut;
layout(push_constant) uniform PC { float dt; vec3 gridMin; float dx; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 idx = ivec3(gl_GlobalInvocationID);
    vec3  x   = pc.gridMin + (vec3(idx) + 0.5) * pc.dx;
    vec3  v   = texture(velField, (x - pc.gridMin) / (pc.dx * GRID_SIZE)).xyz;
    vec3  xp  = x - v * pc.dt;  // back-trace particle position
    vec3  uvw = (xp - pc.gridMin) / (pc.dx * GRID_SIZE);
    imageStore(phiOut, idx, vec4(texture(phiIn, uvw).r));
}
```

### 61.2 SDF Reinitialization (Redistancing)

After advection, φ is no longer a true SDF (|∇φ| ≠ 1). Sussman et al. (1994) solve:

∂φ/∂τ = sign(φ₀)(1 − |∇φ|)

until steady state (|∇φ| ≈ 1 near the zero crossing). GPU implementation: one compute dispatch per pseudo-time step τ, applying the PDE only within a narrow band of ±k·dx around φ=0:

```glsl
// ls_reinit.comp — Sussman redistancing, narrow-band
layout(set=0,binding=0) readonly buffer PhiIn  { float phiIn[];  };
layout(set=0,binding=1) writeonly buffer PhiOut { float phiOut[]; };
layout(push_constant) uniform PC { float dtau; float dx; float bandWidth; } pc;
layout(local_size_x=64) in;

float godunovGrad(uint v) {
    float dxp = (phiIn[v+1]        - phiIn[v]) / pc.dx;
    float dxm = (phiIn[v]          - phiIn[v-1]) / pc.dx;
    float dyp = (phiIn[v+GRID_X]   - phiIn[v]) / pc.dx;
    float dym = (phiIn[v]          - phiIn[v-GRID_X]) / pc.dx;
    float dzp = (phiIn[v+GRID_XY]  - phiIn[v]) / pc.dx;
    float dzm = (phiIn[v]          - phiIn[v-GRID_XY]) / pc.dx;
    float sp   = sign(phiIn[v]);
    float Dp = (sp > 0.0)
        ? sqrt(max(dxm,0)*max(dxm,0)+max(-dxp,0)*max(-dxp,0)+max(dym,0)*max(dym,0)+max(-dyp,0)*max(-dyp,0)+max(dzm,0)*max(dzm,0)+max(-dzp,0)*max(-dzp,0))
        : sqrt(max(-dxm,0)*max(-dxm,0)+max(dxp,0)*max(dxp,0)+max(-dym,0)*max(-dym,0)+max(dyp,0)*max(dyp,0)+max(-dzm,0)*max(-dzm,0)+max(dzp,0)*max(dzp,0));
    return Dp;
}

void main() {
    uint v = gl_GlobalInvocationID.x;
    if (abs(phiIn[v]) > pc.bandWidth) { phiOut[v] = phiIn[v]; return; }
    float S = sign(phiIn[v]);
    phiOut[v] = phiIn[v] - pc.dtau * S * (godunovGrad(v) - 1.0);
}
```

[Source: Osher & Sethian "Fronts Propagating with Curvature-Dependent Speed", J. Comput. Phys. 1988; Sussman et al. "A Level Set Approach for Computing Solutions to Incompressible Two-Phase Flow", J. Comput. Phys. 1994]

### 61.3 GPU Fast Marching Method (FMM)

FMM builds the SDF outward from an initialised narrow band using a heap-like wavefront. GPU parallelism: Jeong & Whitaker (2008) parallel FMM tiles each grid block and uses a local heap per workgroup, merging wavefronts at block boundaries:

```glsl
// fmm_sweep.comp — one FMM sweep pass (Godunov upwind stencil)
layout(set=0,binding=0) buffer SDF    { float sdf[]; };
layout(set=0,binding=1) buffer State  { uint  state[]; };  // 0=unknown,1=trial,2=known
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    uint v = flatIdx(ivec3(gl_GlobalInvocationID));
    if (state[v] == 2u) return;  // already frozen
    // Godunov upwind: take minimum of ±x, ±y, ±z known neighbours
    float ax = min(sdf[v-1], sdf[v+1]);
    float ay = min(sdf[v-GRID_X], sdf[v+GRID_X]);
    float az = min(sdf[v-GRID_XY], sdf[v+GRID_XY]);
    // solve quadratic (ax-t)²+(ay-t)²+(az-t)²=dx² for t
    // ... Godunov quadratic solve ...
    float t = godunovSolve(ax, ay, az, DX);
    if (t < sdf[v]) { sdf[v] = t; state[v] = 1u; }
}
```

Iterate until no trial nodes remain. [Source: Jeong & Whitaker "A Fast Iterative Method for Eikonal Equations", SIAM J. Sci. Comput. 2008]

### 61.4 Narrow-Band Level Sets

Only voxels within a band of width ±W·dx of φ=0 need updating. Compact active voxels with prefix scan (§52 dirty-bitmask pattern) before each sweep to avoid dispatching over the full grid. For a 256³ grid with W=3, this limits active voxels to ~5% of the grid at typical surface areas.

---

### 62. Swept Volumes and Minkowski Sums

*Audience: systems developers, graphics application developers.*

The swept volume of an object O moving along a trajectory T is the union of all instantaneous placements O(t). GPU computation is used in robot workspace analysis, CNC toolpath collision avoidance, and dynamic broad-phase proximity queries (§50.1 SAP needs swept AABBs for fast-moving objects).

### 62.1 Swept AABB for Broadphase

For a rigid body moving from pose A to pose B in one frame, the swept AABB bounds all intermediate positions. Compute per-object:

```glsl
// swept_aabb.comp
layout(set=0,binding=0) readonly buffer ObjOld { AABB old_aabb[]; };
layout(set=0,binding=1) readonly buffer ObjNew { AABB new_aabb[]; };
layout(set=0,binding=2) writeonly buffer Swept  { AABB swept[]; };
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    swept[i].min = min(old_aabb[i].min, new_aabb[i].min);
    swept[i].max = max(old_aabb[i].max, new_aabb[i].max);
}
```

Feed swept AABBs into the SAP broadphase (§50.1) for CCD (Continuous Collision Detection).

### 62.2 Minkowski Sum of Convex Polyhedra

The Minkowski sum A ⊕ B has vertices at {a + b : a ∈ A, b ∈ B}. For convex polytopes, the exact sum is computed by merging the Gauss maps: sort all face normals of A and B on the Gauss sphere, merge-sort by angle, emit a new face for each angular range. GPU parallelism on the Gauss map merge:

```glsl
// gauss_map_sort.comp — sort face normals by longitude on Gauss sphere
layout(set=0,binding=0) readonly buffer FaceNormals { vec4 normals[]; };  // xyz normal, w=faceID
layout(set=0,binding=1) writeonly buffer Sorted { vec4 sorted[]; };
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    float lon = atan(normals[i].y, normals[i].x);  // longitude for Gauss map sort
    sorted[i] = vec4(normals[i].xyz, lon);
}
// Follow with GPU radix sort on w (longitude), then CPU merge of A and B sorted arrays
```

For non-convex objects: decompose via V-HACD (§42 Table A) then sum each convex part pair and union the resulting meshes. [Source: Dobkin et al. "Computing the Minkowski Sum of Convex Polytopes", Discrete & Computational Geometry 1993]

### 62.3 CNC Toolpath Swept Volume via SDF Dilation

In CNC verification, the swept volume of a tool along a path equals the SDF of the path dilated by the tool radius. GPU implementation: rasterise the toolpath as a polyline SDF (JFA, §3.4) then dilate by the tool radius as a post-process (morphological dilation on 3D SDF voxel grid):

```glsl
// sdf_dilate.comp — Minkowski dilation by radius r (offset SDF)
layout(set=0,binding=0, r32f) uniform readonly  image3D sdfIn;
layout(set=0,binding=1, r32f) uniform writeonly image3D sdfOut;
layout(push_constant) uniform PC { float radius; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 v   = ivec3(gl_GlobalInvocationID);
    float val = imageLoad(sdfIn, v).r - pc.radius;  // offset = dilation
    imageStore(sdfOut, v, vec4(val));
}
```

The Minkowski sum A ⊕ B_sphere(r) equals offsetting A's SDF by −r everywhere — O(1) per voxel. [Source: Jones et al. "3D Distance Fields: A Survey of Techniques and Applications", TVCG 2006]

---

### 63. Volumetric 3D Reconstruction (TSDF Fusion)

*Audience: systems developers, graphics application developers.*

Truncated Signed Distance Function (TSDF) fusion (Newcombe et al. "KinectFusion" 2011) integrates a stream of depth images into a persistent 3D SDF volume on the GPU in real time. Each new depth frame updates the TSDF; GPU marching cubes extracts the current surface mesh on demand.

### 63.1 Depth Map Ray Casting and TSDF Update

For each depth pixel (u,v) with depth z, back-project into world space, then update all voxels along the ray within truncation distance μ:

```glsl
// tsdf_integrate.comp — integrate one depth frame into TSDF volume
layout(set=0,binding=0) buffer TSDF  { float tsdf[];   };  // current SDF value
layout(set=0,binding=1) buffer Weight{ float weight[]; };  // running sum of weights
layout(set=0,binding=2) uniform sampler2D depthImage;
layout(push_constant) uniform PC {
    mat4 T_world_cam;    // camera pose in world
    mat4 K_inv;          // inverse camera intrinsics
    vec3 gridMin; float dx;
    ivec3 gridDim; float mu;   // truncation distance
} pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 vox = ivec3(gl_GlobalInvocationID);
    vec3  wp  = pc.gridMin + (vec3(vox)+0.5)*pc.dx;
    // Transform world point into camera space
    vec4  cp  = inverse(pc.T_world_cam) * vec4(wp, 1.0);
    if (cp.z <= 0.0) return;
    // Project to pixel
    vec3  pp  = (inverse(mat3(pc.K_inv)) * cp.xyz) / cp.z;
    vec2  uv  = pp.xy / vec2(IMAGE_W, IMAGE_H);
    if (any(lessThan(uv,vec2(0))) || any(greaterThan(uv,vec2(1)))) return;
    float Dmeas = texture(depthImage, uv).r;
    if (Dmeas <= 0.0) return;
    float sdf_val = Dmeas - cp.z;   // signed distance along ray
    if (sdf_val < -pc.mu) return;   // behind surface, too far
    float tsdf_val = clamp(sdf_val / pc.mu, -1.0, 1.0);
    uint  flat     = vox.z*pc.gridDim.y*pc.gridDim.x + vox.y*pc.gridDim.x + vox.x;
    float w        = 1.0;
    tsdf[flat]   = (tsdf[flat] * weight[flat] + tsdf_val * w) / (weight[flat] + w);
    weight[flat] += w;
}
```

[Source: Newcombe et al. "KinectFusion: Real-Time Dense Surface Mapping and Tracking", ISMAR 2011]

### 63.2 Camera Pose Tracking via ICP on TSDF Normal Field

Each incoming depth frame is aligned to the current TSDF by ICP (§87.3) against the TSDF surface normal field. Project current TSDF normals to a normal map, then run point-to-plane ICP between the new depth frame's point cloud and the TSDF normal map:

```glsl
// tsdf_normal.comp — extract surface normal from TSDF gradient
layout(set=0,binding=0) readonly buffer TSDF { float tsdf[]; };
layout(set=0,binding=1, rgba16f) uniform writeonly image3D normalVol;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 v = ivec3(gl_GlobalInvocationID);
    // Central difference gradient of TSDF → surface normal
    float dx = tsdf[flat(v+ivec3(1,0,0))] - tsdf[flat(v-ivec3(1,0,0))];
    float dy = tsdf[flat(v+ivec3(0,1,0))] - tsdf[flat(v-ivec3(0,1,0))];
    float dz = tsdf[flat(v+ivec3(0,0,1))] - tsdf[flat(v-ivec3(0,0,1))];
    imageStore(normalVol, v, vec4(normalize(vec3(dx,dy,dz)), 0.0));
}
```

### 63.3 Mesh Extraction from TSDF

Run GPU marching cubes (§3.2) on the TSDF volume at the zero crossing. For real-time KinectFusion the MC runs every frame on a 256³ grid in ~2 ms (RTX 3070); for offline processing a 512³ grid runs every 10th frame.

```glsl
// tsdf_mc.comp — marching cubes on TSDF (reuses §3.2 pattern)
// Sample tsdf[] at 8 cube corners; look up MC edge table; emit triangles.
// Vertices interpolated at zero crossing: v = lerp(a,b, tsdf[a]/(tsdf[a]-tsdf[b]))
```

[Source: Newcombe et al. 2011; Steinbrücker et al. "Real-Time Visual Odometry from Dense RGB-D Images", ICCVW 2011; open-source reference: Open3D KinectFusion https://www.open3d.org]

### 63.4 Voxel Hashing for Unbounded Reconstruction

The 256³ bounded grid of basic KinectFusion limits scene size. Voxel hashing (Nießner et al. 2013) stores only occupied voxel blocks in a GPU hash table:

```glsl
// vhashing_lookup.comp — voxel hash table lookup/insert
layout(set=0,binding=0) buffer HashTable { HashEntry table[HASH_SIZE]; };
uint hashKey(ivec3 blockCoord) {
    return (blockCoord.x*73856093u ^ blockCoord.y*19349663u ^ blockCoord.z*83492791u)
           % HASH_SIZE;
}
// On insert: linear probing with atomicCompSwap on entry.key
// On lookup: follow probe chain until key match or empty slot
```

[Source: Nießner et al. "Real-time 3D Reconstruction at Scale Using Voxel Hashing", SIGGRAPH Asia 2013]

---

### 64. SDF-Based Collision Detection

*Audience: graphics application developers, systems developers.*

§24 covered BVH/GJK collision for convex polyhedra. SDF-based collision detection uses the signed distance field representation directly: query the SDF at the penetrating point for depth and gradient direction, or run GJK over SDF-defined primitives. This is the standard approach for deformable body–rigid body coupling, character–environment collision, and SDF CSG contact.

### 64.1 Point-SDF Penetration Query

For a rigid body with vertices {vᵢ}, query each vertex against the scene SDF. If φ(vᵢ) < 0, the point is inside; depth = |φ(vᵢ)|, normal = ∇φ at vᵢ:

```glsl
// sdf_contact.comp — generate contacts for rigid body vertices vs scene SDF
layout(set=0,binding=0) readonly buffer Vertices { vec3 verts[]; };
layout(set=0,binding=1) uniform sampler3D sceneSDF;
layout(set=0,binding=2) buffer Contacts { ContactPoint contacts[]; };
layout(set=0,binding=3) buffer ContactCount { uint count; };
layout(push_constant) uniform PC { vec3 sdfMin; float sdfDx; mat4 bodyTransform; } pc;
layout(local_size_x=64) in;
void main() {
    uint i    = gl_GlobalInvocationID.x;
    vec3 wp   = (pc.bodyTransform * vec4(verts[i],1.0)).xyz;
    vec3 uvw  = (wp - pc.sdfMin) / (pc.sdfDx * SDF_DIM);
    if (any(lessThan(uvw,vec3(0)))||any(greaterThan(uvw,vec3(1)))) return;
    float phi = texture(sceneSDF, uvw).r;
    if (phi < 0.0) {
        // Normal from central-difference gradient of SDF
        vec3 n = normalize(vec3(
            textureOffset(sceneSDF,uvw,ivec3(1,0,0)).r - textureOffset(sceneSDF,uvw,ivec3(-1,0,0)).r,
            textureOffset(sceneSDF,uvw,ivec3(0,1,0)).r - textureOffset(sceneSDF,uvw,ivec3(0,-1,0)).r,
            textureOffset(sceneSDF,uvw,ivec3(0,0,1)).r - textureOffset(sceneSDF,uvw,ivec3(0,0,-1)).r
        ));
        uint slot = atomicAdd(count, 1u);
        contacts[slot] = ContactPoint(wp, n, -phi, i);
    }
}
```

[Source: Müller et al. "Unified Particle Physics for Real-Time Applications", SIGGRAPH 2014; Macklin et al. "XPBD: Position-Based Simulation of Compliant Constrained Dynamics", MIG 2016]

### 64.2 SDF-SDF Collision via Gradient Descent

When both objects are represented as SDFs, find the closest pair of surface points by gradient-following one SDF's zero-crossing in the other's field:

```glsl
// sdf_sdf_closest.comp — Newton iterate for SDF-SDF closest point
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    vec3 p  = candidatePoint[i];  // initial seed near boundary
    for (int iter=0; iter<MAX_ITER; iter++) {
        float phiA = sampleSDF(sdfA, p);
        float phiB = sampleSDF(sdfB, p);
        vec3  nA   = sdfGradient(sdfA, p);
        vec3  nB   = sdfGradient(sdfB, p);
        if (abs(phiA) < TOL && abs(phiB) < TOL) break;
        // Step toward zero crossing of both SDFs simultaneously
        p -= (phiA * nA + phiB * nB) * 0.5;
    }
    closestPoint[i] = p;
    penetrationDepth[i] = -min(sampleSDF(sdfA,p), sampleSDF(sdfB,p));
}
```

### 64.3 SDF-Based Character–Terrain Collision

For character capsule vs heightfield terrain (§71.4), the SDF approach is exact and fast. Evaluate the terrain SDF at the capsule centre and hemispheres; push the capsule out by the penetration depth along the SDF gradient:

```glsl
// capsule_terrain.comp
layout(local_size_x=64) in;
void main() {
    uint c   = gl_GlobalInvocationID.x;
    vec3 base= capsule[c].base, tip = capsule[c].tip;
    float r  = capsule[c].radius;
    // Sample terrain SDF at 3 points along capsule axis
    for (int k=0; k<=2; k++) {
        vec3 p    = mix(base, tip, float(k)*0.5);
        float phi = terrainSDF(p);
        if (phi < r) {
            vec3 n = terrainNormal(p);
            float pen = r - phi;
            capsule[c].base += n * pen;
            capsule[c].tip  += n * pen;
        }
    }
}
```

[Source: Erleben et al. "Physics-Based Animation", Charles River Media 2005; Macklin et al. 2014]

### 64.4 EPA: Expanding Polytope Algorithm for SDF Primitives

When GJK (§24.4) determines that two SDF-defined convex primitives intersect, EPA finds the minimum penetration depth by expanding a polytope outward until it touches the Minkowski difference boundary. Each EPA iteration runs on GPU for independent object pairs:

```glsl
// epa_step.comp — one EPA polytope expansion step
layout(set=0,binding=0) buffer Polytope { vec3 verts[]; uint nVerts; };
layout(set=0,binding=1) buffer EPAFaces  { uvec3 faces[]; float dist[]; uint nFaces; };
layout(local_size_x=1) in;
void main() {
    // Find closest face (minimum distance to origin)
    uint best = 0;
    for (uint f=1; f<EPAFaces.nFaces; f++)
        if (dist[f] < dist[best]) best=f;
    // Support point of Minkowski diff in face normal direction
    vec3 n    = faceNormal(faces[best]);
    vec3 supp = supportSDF_A(n) - supportSDF_B(-n);
    if (dot(supp,n) - dist[best] < EPA_TOL) { penetrationNormal=n; penetrationDepth=dist[best]; return; }
    // Expand polytope: add supp, remove faces visible from supp, add horizon faces
    expandPolytope(supp);
}
```

[Source: Bergen "A Fast and Robust GJK Implementation for Collision Detection of Convex Objects", JGTOOLS 1999]

---

### 65. Isosurface Extraction

*Audience: graphics application developers, systems developers.*

Isosurface extraction converts a scalar field φ(x,y,z) into a triangle or quad mesh at a given isovalue τ. The field may be a density grid (CT/MRI), a level set (§61), a TSDF (§63), or an SDF (§64). Three GPU-native algorithms exist: Marching Cubes, Dual Contouring, and Surface Nets.

### 65.1 Marching Cubes on GPU

Marching Cubes (Lorensen & Cline 1987) classifies each grid cell by the sign pattern of its 8 corner values, looks up the triangle configuration (256-entry table), and emits triangles. GPU implementation uses a two-pass approach:

```glsl
// mc_classify.comp — pass 1: count triangles per cell
layout(set=0,binding=0) uniform sampler3D phi;  // scalar field
layout(set=0,binding=1) writeonly buffer TriCount { uint nTris[]; };
layout(set=0,binding=2) readonly buffer MCTable  { uint config[256]; };  // tri count table
layout(push_constant) uniform PC { float isoVal; ivec3 dim; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c, pc.dim-1))) return;
    uint  ci = 0u;
    for (int k=0;k<8;k++) {
        ivec3 ofs = ivec3(k&1,(k>>1)&1,(k>>2)&1);
        float v   = texelFetch(phi, c+ofs, 0).r;
        if (v > pc.isoVal) ci |= (1u<<k);
    }
    nTris[flatIdx(c)] = bitCount(config[ci]);  // precomputed triangle count
}
// Pass 2: prefix-sum nTris → offset per cell, then emit vertices via edge interpolation
```

For pass 2, each cell looks up the edge intersection table, interpolates vertex positions on the 12 edges, and writes into a vertex buffer at the prefix-summed offset. [Source: Lorensen & Cline "Marching Cubes: A High Resolution 3D Surface Construction Algorithm", SIGGRAPH 1987; GPU MC: Nguyen "GPU Gems 3 — Chapter 1: Generating Complex Procedural Terrains", 2007]

### 65.2 Dual Contouring

Dual Contouring (Ju et al. 2002) places a vertex inside each active cell (one that straddles the isosurface) rather than on edges — producing sharp features by solving a quadratic error function (QEF) from the edge intersection normals:

```glsl
// dc_qef.comp — solve QEF per active cell to find dual vertex
layout(set=0,binding=0) readonly buffer Edges { EdgeIntersection ei[]; };
layout(set=0,binding=1) writeonly buffer DualVerts { vec3 dv[]; };
layout(local_size_x=64) in;
void main() {
    uint c  = gl_GlobalInvocationID.x;
    // Collect edge intersections for this cell (up to 12)
    vec3 A[12]; vec3 n[12]; int m=0;
    for (int e=0; e<12; e++) {
        EdgeIntersection x = getCellEdge(ei, c, e);
        if (x.active) { A[m]=x.pos; n[m]=x.normal; m++; }
    }
    if (m==0) { dv[c]=vec3(0); return; }
    // QEF: minimize sum_i (nᵢᵀ(x-Aᵢ))² — solve 3×3 normal equation
    mat3 ATA = mat3(0); vec3 ATb = vec3(0);
    for (int i=0;i<m;i++) {
        ATA += outerProduct(n[i],n[i]);
        ATb += dot(n[i],A[i]) * n[i];
    }
    dv[c] = clampToCell(inverse(ATA+1e-4*mat3(1))*ATb, c);
}
```

[Source: Ju et al. "Dual Contouring of Hermite Data", SIGGRAPH 2002; Schaefer & Warren "Dual Contouring: The Secret Sauce", 2002]

### 65.3 Surface Nets

Surface Nets (Gibson 1998) places a vertex at the centroid of all active edge intersection points in a cell — a simpler, smoother alternative to Dual Contouring, GPU-parallelizable per cell:

```glsl
// surface_nets.comp — place vertex at centroid of edge intersections
layout(set=0,binding=0) uniform sampler3D phi;
layout(set=0,binding=1) writeonly buffer SNVerts { vec3 snv[]; uint active[]; };
layout(push_constant) uniform PC { float isoVal; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    vec3  sum = vec3(0.0); int cnt = 0;
    // 12 edges: check sign change, interpolate crossing position
    for (int e=0; e<12; e++) {
        ivec3 a = c + EDGE_A[e], b = c + EDGE_B[e];
        float va = texelFetch(phi,a,0).r, vb = texelFetch(phi,b,0).r;
        if ((va < pc.isoVal) != (vb < pc.isoVal)) {
            float t = (pc.isoVal - va) / (vb - va);
            sum += mix(vec3(a), vec3(b), t);
            cnt++;
        }
    }
    if (cnt > 0) { snv[flatIdx(c)] = sum/float(cnt); active[flatIdx(c)] = 1u; }
    else           active[flatIdx(c)] = 0u;
}
// Quad faces generated by connecting dual vertices of adjacent active cells
```

[Source: Gibson "Constrained Elastic Surface Nets: Generating Smooth Surfaces from Binary Segmented Data", MICCAI 1998; Naive Surface Nets implementation: https://0fps.net/2012/07/12/smooth-voxel-terrain/]

### 65.4 Adaptive Isosurface via SVO

For large sparse volumes, run marching cubes only on leaf nodes of the sparse voxel octree (§27.3) that straddle the isosurface. The SVO traversal is a BFS over active nodes:

```glsl
// mc_svo.comp — dispatch MC only on active SVO leaves near isosurface
layout(set=0,binding=0) readonly buffer SVONodes  { SVONode nodes[]; };
layout(set=0,binding=1) writeonly buffer ActiveLeaves { uint leaves[]; };
layout(set=0,binding=2) buffer LeafCount { uint count; };
layout(local_size_x=64) in;
void main() {
    uint n = gl_GlobalInvocationID.x;
    if (!nodes[n].isLeaf) return;
    // Node straddles isosurface if min and max corner values differ in sign
    if (nodes[n].phiMin < 0.0 && nodes[n].phiMax > 0.0) {
        uint slot = atomicAdd(count, 1u);
        leaves[slot] = n;
    }
}
// vkCmdDispatchIndirect with count → MC pass 1 (§65.1) over leaves only
```

[Source: Kazhdan et al. "Screened Poisson Surface Reconstruction", TOG 2013; SVO MC: Crassin & Green "Octree-Based Sparse Voxelization", GPU Pro 4, 2013]

---

### 66. GPU Ambient Occlusion

*Audience: graphics application developers.*

Ambient occlusion quantifies how much of the hemisphere above a surface point is occluded by nearby geometry — a geometric signal, not a lighting model. Four GPU algorithm families span the quality/performance spectrum: SSAO (screen-space), HBAO+ (horizon-based), GTAO (ground-truth AO), and precomputed baked AO (ray traced).

### 66.1 SSAO: Screen-Space Ambient Occlusion

SSAO (Crytek 2007) samples random positions in a hemisphere around each G-buffer point, tests each against the depth buffer, and counts occlusions:

```glsl
// ssao.comp — screen-space ambient occlusion
layout(set=0,binding=0) uniform sampler2D gDepth;
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2) uniform sampler2D noiseTexture;  // 4×4 random rotation vectors
layout(set=0,binding=3) readonly buffer SSAOKernel { vec3 kernel[N_SAMPLES]; };
layout(set=0,binding=4, r8) uniform writeonly image2D aoOut;
layout(push_constant) uniform PC { mat4 proj; mat4 invProj; float radius; float bias; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  n    = texelFetch(gNormal, pix, 0).xyz;
    float d    = texelFetch(gDepth, pix, 0).r;
    vec3  pos  = viewSpacePos(pix, d, pc.invProj);
    // TBN from noise texture rotation
    vec3  randVec = texelFetch(noiseTexture, pix%4, 0).xyz;
    vec3  t    = normalize(randVec - n*dot(randVec,n));
    vec3  b    = cross(n, t);
    mat3  TBN  = mat3(t, b, n);
    float occ  = 0.0;
    for (uint s=0; s<N_SAMPLES; s++) {
        vec3  spos = pos + TBN * kernel[s] * pc.radius;
        vec4  clip = pc.proj * vec4(spos,1.0);
        vec2  suv  = clip.xy/clip.w*0.5+0.5;
        float sd   = texture(gDepth, suv).r;
        float sz   = viewSpaceZ(sd, pc.invProj);
        float rangeCheck = smoothstep(0.0,1.0,pc.radius/abs(pos.z-sz));
        occ += (sz >= spos.z + pc.bias ? 1.0 : 0.0) * rangeCheck;
    }
    imageStore(aoOut, pix, vec4(1.0 - occ/float(N_SAMPLES)));
}
```

[Source: Mittring "Finding Next Gen — CryEngine 2", SIGGRAPH 2007; Shanmugam & Arikan "Hardware Accelerated Ambient Occlusion Techniques on GPUs", I3D 2007]

### 66.2 HBAO+: Horizon-Based Ambient Occlusion

HBAO (Bavoil & Sainz 2008) marches along multiple screen-space directions from each pixel, finds the maximum elevation angle (horizon), and integrates the sine of the horizon to estimate AO. HBAO+ (NVIDIA 2012) adds interleaved sampling to reduce noise:

```glsl
// hbao.comp — horizon-based AO, one direction step
layout(set=0,binding=0) uniform sampler2D linearDepth;
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2, r8) uniform writeonly image2D aoDir;
layout(push_constant) uniform PC { vec2 focalLen; float radius; float stepSize; int nDir; int nSteps; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(pix)+0.5)/vec2(imageSize(aoDir));
    vec3  P   = reconstructView(uv, texture(linearDepth,uv).r);
    vec3  N   = texture(gNormal, uv).xyz;
    float ao  = 0.0;
    for (int d=0; d<pc.nDir; d++) {
        float angle = float(d)/float(pc.nDir) * 3.14159;
        vec2  dir   = vec2(cos(angle), sin(angle)) / pc.focalLen * pc.stepSize;
        float h1 = -1e9, h2 = -1e9;  // max horizon angles
        for (int s=1; s<=pc.nSteps; s++) {
            vec3 Q1 = reconstructView(uv+dir*float(s), texture(linearDepth,uv+dir*float(s)).r);
            vec3 Q2 = reconstructView(uv-dir*float(s), texture(linearDepth,uv-dir*float(s)).r);
            vec3 v1 = Q1-P, v2 = Q2-P;
            float d1 = length(v1), d2 = length(v2);
            if (d1 < pc.radius) h1 = max(h1, dot(normalize(v1),N)/d1);
            if (d2 < pc.radius) h2 = max(h2, dot(normalize(v2),N)/d2);
        }
        ao += 1.0 - 0.5*(sin(asin(h1))+sin(asin(h2)));
    }
    imageStore(aoDir, pix, vec4(ao/float(pc.nDir)));
}
```

[Source: Bavoil & Sainz "Image-Space Horizon-Based Ambient Occlusion", SIGGRAPH 2008 Sketches; NVIDIA HBAO+ SDK]

### 66.3 GTAO: Ground-Truth Ambient Occlusion

GTAO (Jimenez et al. 2016) corrects the hemisphere integration formula so that the result converges to ray-traced AO as sample count increases — unlike SSAO and HBAO which use ad-hoc approximations. The bent normal (the direction of least occlusion) is also computed:

```glsl
// gtao.comp — ground-truth AO integration (visibility function integral)
// Each slice integrates visibility along a screen-space direction using
// the exact formula: AO = 1 - (1/π) ∫₀^π ∫_h1^h2 cos(θ-γ) dθ dφ
// where γ = projected normal angle, h1/h2 = horizon angles per direction
float integrateArc(float h1, float h2, float n) {
    // Analytical integral of the visibility function
    float sinN = sin(n);
    return (-cos(2*h1-n)+cos(n)+2*h1*sinN - (-cos(2*h2-n)+cos(n)+2*h2*sinN)) * 0.25;
}
// main: for each direction slice, compute h1, h2, call integrateArc, accumulate
```

[Source: Jimenez et al. "Practical Real-Time Strategies for Accurate Indirect Occlusion", SIGGRAPH 2016; XeGTAO implementation: https://github.com/GameTechDev/XeGTAO]

### 66.4 Ray-Traced AO and Bent Normals

With `VK_KHR_ray_query`, AO becomes a direct ray cast from each G-buffer point into the hemisphere, returning a binary hit/miss:

```glsl
// rtao.comp — ray-traced ambient occlusion via ray query
#extension GL_EXT_ray_query : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) uniform sampler2D gWorldPos;
layout(set=0,binding=2) uniform sampler2D gNormal;
layout(set=0,binding=3, r16f) uniform writeonly image2D aoOut;
layout(set=0,binding=4) readonly buffer Halton { vec2 halton[]; };
layout(push_constant) uniform PC { uint frameIdx; float radius; int nRays; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  P   = texelFetch(gWorldPos, pix, 0).xyz;
    vec3  N   = texelFetch(gNormal,   pix, 0).xyz;
    float ao  = 0.0;
    uint seed = pix.y*WIDTH+pix.x + pc.frameIdx*WIDTH*HEIGHT;
    for (int r=0; r<pc.nRays; r++) {
        vec3 dir = cosineSampleHemisphere(N, halton[seed%1024+r]);
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT,
            0xFF, P+N*1e-3, 0.0, dir, pc.radius);
        rayQueryProceedEXT(rq);
        ao += (rayQueryGetIntersectionTypeEXT(rq,true)
               == gl_RayQueryCommittedIntersectionNoneEXT) ? 1.0 : 0.0;
    }
    imageStore(aoOut, pix, vec4(ao/float(pc.nRays)));
}
```

[Source: Stachowiak "Stochastic All The Things: Raytracing in Hybrid Real-Time Rendering", Digital Dragons 2018]

---

### 67. Volumetric Ray Marching

*Audience: graphics application developers.*

Volumetric rendering integrates light scattering and absorption through a participating medium — clouds, fog, smoke, fire, subsurface scattering in skin. The GPU algorithm is ray marching through a 3D volume from the near plane to the far plane or until the accumulated opacity reaches 1. This is distinct from TSDF (§63, geometry reconstruction) and level sets (§61, surface evolution).

### 67.1 Basic Ray Marching

Ray march from the eye through a 3D density volume, accumulating scattered radiance and transmitted light:

```glsl
// raymarch.frag — single-pass volumetric ray march
uniform sampler3D density;  // 3D density field
uniform sampler3D temperature;  // for fire/emission
uniform vec3 lightDir; uniform vec3 lightColor;
uniform float stepSize; uniform int maxSteps;
uniform float sigmaA; uniform float sigmaS;  // absorption and scattering coefficients
in vec3 rayOrigin, rayDir;
void main() {
    vec3  P   = rayOrigin;
    float T   = 1.0;  // transmittance
    vec3  L   = vec3(0.0);
    for (int i=0; i<maxSteps && T > 0.01; i++) {
        P += rayDir * stepSize;
        if (!inBounds(P)) break;
        vec3  uvw = worldToUVW(P);
        float rho = texture(density, uvw).r;
        if (rho < 1e-5) continue;
        float sigT = (sigmaA + sigmaS) * rho;
        float dT   = exp(-sigT * stepSize);
        // Shadow ray (simplified: single shadow sample)
        float shadow = shadowMarch(P, lightDir, density, sigT);
        // Phase function (Henyey-Greenstein)
        float cosTheta = dot(rayDir, -lightDir);
        float phase = henyeyGreenstein(cosTheta, 0.6);
        vec3  emit  = texture(temperature, uvw).r > 0.5 ?
                      blackbody(texture(temperature,uvw).r * 2000.0) : vec3(0.0);
        L  += T * (1.0-dT) * (lightColor * phase * shadow * sigmaS + emit);
        T  *= dT;
    }
    fragColor = vec4(L, 1.0-T);
}
```

[Source: Wrenninge & Bin Zafar "Production Volume Rendering", SIGGRAPH 2011; Fong et al. "Production Volume Rendering: Design and Implementation", SIGGRAPH 2017 Course]

### 67.2 Froxel Volumetrics

Froxels (frustum voxels, Hillaire 2016) discretize the camera frustum into a 3D grid in clip space — typically 160×90×64. A compute shader fills density/scattering into the froxel grid, then ray marches through it in-place:

```glsl
// froxel_fill.comp — populate froxel density from world-space volumes
layout(set=0,binding=0, rgba16f) uniform image3D froxelGrid;  // R=density G=emission B=scatter
layout(push_constant) uniform PC { mat4 invViewProj; vec3 camPos; float nearZ; float farZ; int D; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=4) in;
void main() {
    ivec3 fr  = ivec3(gl_GlobalInvocationID);
    vec3  ndc = vec3(vec2(fr.xy)/vec2(160,90)*2.0-1.0,
                     froxelDepth(fr.z, pc.nearZ, pc.farZ, pc.D));
    vec4  wp  = pc.invViewProj * vec4(ndc,1.0);
    wp /= wp.w;
    // Evaluate all registered volume primitives at world position wp.xyz
    float density = evalVolumes(wp.xyz);  // sum of Gaussian/exponential fog volumes
    imageStore(froxelGrid, fr, vec4(density, emissionAt(wp.xyz), scatterAt(wp.xyz), 0.0));
}
// Second pass: ray march through froxel grid (128 cells per screen-space ray)
```

[Source: Hillaire "Physically Based Sky, Atmosphere and Cloud Rendering in Frostbite", SIGGRAPH 2016; Wronski "Volumetric Fog: Unified, Compute Shader Based Solution to Atmospheric Scattering", ACM Digital Library 2014]

### 67.3 Atmospheric Scattering

Physically based sky rendering (Bruneton & Neyret 2008) precomputes multiple-scattering LUTs (transmittance, scattering, irradiance) and samples them at runtime for sky colour and aerial perspective. The precomputation runs as a series of 2D/3D compute passes:

```glsl
// atmos_transmittance.comp — precompute transmittance LUT T(h, μ)
// h = altitude, μ = cos(view-zenith angle)
layout(set=0,binding=0, rg16f) uniform writeonly image2D transLUT;
layout(push_constant) uniform PC { float Rearth; float Ratmos; float Hr; float Hm; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 id = ivec2(gl_GlobalInvocationID.xy);
    float h  = pc.Rearth + float(id.y)/LUT_H * (pc.Ratmos-pc.Rearth);
    float mu = float(id.x)/LUT_W * 2.0 - 1.0;
    // Integrate optical depth along view ray from (h,μ) to atmosphere boundary
    float optR = 0.0, optM = 0.0;
    float ds   = rayLengthAtmos(h, mu, pc.Rearth, pc.Ratmos) / STEPS;
    for (int s=0; s<STEPS; s++) {
        float r = length(vec2(h,0)+vec2(mu,sqrt(1-mu*mu))*(s+0.5)*ds);
        optR += exp(-(r-pc.Rearth)/pc.Hr) * ds;
        optM += exp(-(r-pc.Rearth)/pc.Hm) * ds;
    }
    imageStore(transLUT, id, vec4(exp(-betaR*optR-betaM*optM*1.1)));
}
```

[Source: Bruneton & Neyret "Precomputed Atmospheric Scattering", CGF 2008; Hillaire "A Scalable and Production Ready Sky and Atmosphere Rendering Technique", EG 2020]

### 67.4 VDB Volume Traversal for Rendering

OpenVDB / NanoVDB sparse volumes (§27 SVO ancestry) are traversed for rendering via DDA (Digital Differential Analyzer) through the active voxel tree:

```glsl
// nanovdb_march.comp — ray march through NanoVDB sparse volume
// NanoVDB buffer is uploaded as an SSBO; tree traversal uses the NanoVDB C++ API
// compiled via glslang extension or CUDA interop
layout(set=0,binding=0) readonly buffer NVDBBuffer { uint nvdb[]; };
layout(set=0,binding=1, rgba16f) uniform writeonly image2D renderOut;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  ro   = cameraPos;
    vec3  rd   = normalize(cameraRay(pix));
    float T    = 1.0;  vec3 L = vec3(0.0);
    // DDA traverse: skip empty nodes in O(log N) per step
    NVDBAccessor acc = createAccessor(nvdb);
    float t = 0.0;
    while (t < MAX_DIST && T > 0.01) {
        vec3  p  = ro + rd*t;
        float rho= nvdbSampleFloat(acc, p);
        float dt = rho > 0.0 ? DENSE_STEP : nextEmptyBoundary(acc, p, rd);
        if (rho > 1e-5) {
            float dT = exp(-rho * dt * SIGMA_T);
            L += T*(1-dT)*scatterLight(p, rd);
            T *= dT;
        }
        t += dt;
    }
    imageStore(renderOut, pix, vec4(L, 1.0-T));
}
```

[Source: Museth "NanoVDB: A GPU-Friendly and Portable VDB Data Structure for Real-Time Rendering and Simulation", SIGGRAPH 2021; https://github.com/AcademySoftwareFoundation/openvdb]

---

### 68. GPU Sparse Narrow-Band Level-Set Evolution

*Audience: systems developers, graphics application developers.*

Dense level sets (§61) compute on every voxel; the interface occupies only the narrow band |φ| ≤ 3Δx. Sparse narrow-band representations (VDB/OpenVDB §96.1) store only active voxels, enabling much finer grids for the same memory budget. GPU sparse level sets maintain an active list and update only band voxels per timestep.

### 68.1 Active Voxel List Construction

At each timestep, collect all voxels within the narrow band |φ| ≤ k·Δx into a compact active list:

```glsl
// narrowband_collect.comp — collect active voxels into compact list
layout(set=0,binding=0) readonly buffer Phi    { float phi[];  };
layout(set=0,binding=1) buffer ActiveList { uint active[]; uint count; };
layout(push_constant) uniform PC { ivec3 dim; float dx; float bandWidth; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    uint idx = c.z*pc.dim.y*pc.dim.x + c.y*pc.dim.x + c.x;
    if (abs(phi[idx]) <= pc.bandWidth * pc.dx) {
        uint slot = atomicAdd(count, 1u);
        active[slot] = idx;
    }
}
```

[Source: Museth et al. "OpenVDB: An Open Source Data Structure and Toolkit for High-Resolution Volumes", SIGGRAPH 2013; Nielsen & Museth "Dynamic Tubular Grid: An Efficient Data Structure and Algorithms for High Resolution Level Sets", JCGT 2006]

### 68.2 Narrow-Band Reinitialization via Fast Sweeping

Reinitialise the SDF to exact distance values within the narrow band using a fast sweeping / Godunov scheme applied only to active voxels:

```glsl
// narrowband_reinit.comp — Godunov upwind reinitialization (one sweep pass)
layout(set=0,binding=0) readonly buffer ActiveList { uint active[]; uint count; };
layout(set=0,binding=1) buffer Phi { float phi[]; };
layout(push_constant) uniform PC { ivec3 dim; float dx; int sweep; } pc;
layout(local_size_x=64) in;
void main() {
    uint li  = gl_GlobalInvocationID.x;
    if (li >= count) return;
    uint idx = active[li];
    ivec3 c  = unflatten(idx, pc.dim);
    // Godunov upwind: pick upwind neighbour in each axis
    float px = minNeighbour(phi, c, ivec3(1,0,0), pc.dim, pc.sweep);
    float py = minNeighbour(phi, c, ivec3(0,1,0), pc.dim, pc.sweep);
    float pz = minNeighbour(phi, c, ivec3(0,0,1), pc.dim, pc.sweep);
    float d  = solveGodunovEikonal(px, py, pz, pc.dx);
    phi[idx] = sign(phi[idx]) * min(abs(phi[idx]), d);
}
```

[Source: Zhao "A Fast Sweeping Method for Eikonal Equations", Mathematics of Computation 2004; Bridson "Fluid Simulation for Computer Graphics", 2008 §3]

### 68.3 Level-Set Advection on Active Voxels

Advect the level set under a velocity field using a 5th-order WENO scheme for the spatial derivative and TVD-RK3 for time integration, applied only to active voxels:

```glsl
// narrowband_advect.comp — WENO5 spatial derivative, Euler time step on active voxels
layout(set=0,binding=0) readonly buffer Vel      { vec3 vel[];  };
layout(set=0,binding=1) readonly buffer PhiIn    { float phiIn[]; };
layout(set=0,binding=2) writeonly buffer PhiOut  { float phiOut[]; };
layout(set=0,binding=3) readonly buffer ActiveList { uint active[]; uint count; };
layout(push_constant) uniform PC { ivec3 dim; float dx; float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint li  = gl_GlobalInvocationID.x;
    if (li >= count) return;
    uint idx = active[li];
    ivec3 c  = unflatten(idx, pc.dim);
    vec3  v  = vel[idx];
    // WENO5 upwind: select direction per-axis based on sign of velocity
    float dpx = (v.x>0) ? weno5Minus(phiIn, c, ivec3(1,0,0), pc.dim, pc.dx)
                        : weno5Plus (phiIn, c, ivec3(1,0,0), pc.dim, pc.dx);
    float dpy = (v.y>0) ? weno5Minus(phiIn, c, ivec3(0,1,0), pc.dim, pc.dx)
                        : weno5Plus (phiIn, c, ivec3(0,1,0), pc.dim, pc.dx);
    float dpz = (v.z>0) ? weno5Minus(phiIn, c, ivec3(0,0,1), pc.dim, pc.dx)
                        : weno5Plus (phiIn, c, ivec3(0,0,1), pc.dim, pc.dx);
    phiOut[idx] = phiIn[idx] - pc.dt * (v.x*dpx + v.y*dpy + v.z*dpz);
}
```

[Source: Osher & Fedkiw "Level Set Methods and Dynamic Implicit Surfaces", Springer 2003; Jiang & Peng "Weighted ENO Schemes on Triangular Meshes", JCP 2000]

### 68.4 Topology Change and Band Rebuild

After each advection step, check whether previously inactive voxels have entered the band (zero crossing propagation) and add them to the active list:

```glsl
// narrowband_expand.comp — add newly active voxels (|phi|<band) to active list
layout(set=0,binding=0) readonly buffer Phi { float phi[]; };
layout(set=0,binding=1) buffer ActiveList { uint active[]; uint count; };
layout(set=0,binding=2) buffer ActiveMask { uint mask[];  };  // 1 if currently active
layout(push_constant) uniform PC { ivec3 dim; float dx; float band; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    uint idx = flatIdx(c,pc.dim);
    if (mask[idx]!=0u) return;  // already active
    if (abs(phi[idx]) <= pc.band*pc.dx) {
        mask[idx]=1u;
        uint slot=atomicAdd(count,1u);
        active[slot]=idx;
    }
}
```

[Source: Museth et al. 2013; Houston et al. "A Unified Particle/Grid Fluid Simulation Algorithm", Proceedings of Graphics Interface 2006]

---


---


## Integrations

**Ch208** (this series) covers the surface geometry (metaballs, SDF CSG, Marching Cubes) that is the geometric input and output representation for the SDF volumetric methods in Cat VII. **Ch209** (this series) covers the spatial data structures (BVH, hash grids) that provide the broadphase query layer for the physics simulation in Cat VI. **Ch206** (Shader Algorithm Catalog — Ray Tracing and Procedural) covers the procedural noise and fluid surface rendering techniques that consume the geometry produced here. **Ch133** (Vulkan Compute Queues) covers async compute scheduling for large physics dispatches. **Ch25** (GPU Compute — OpenCL, CUDA, ROCm) covers the compute API landscape that simulation workloads target when Vulkan compute is not the primary runtime.
