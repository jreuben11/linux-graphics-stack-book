# Chapter 243: Object Metadata and Interchange Standards — BIM, STEP, Wikidata, and 3D Object Catalogs (Part XIV)

*Part XIV — Khronos Extended Ecosystem*

**Target audiences**: Graphics application developers evaluating asset pipelines beyond glTF (CAD/BIM import, digital-twin, e-commerce, or cultural-heritage viewers); browser and web platform engineers building 3D catalog or configurator experiences that need structured part/subcomponent metadata, not just a renderable mesh; systems developers assessing interchange-format complexity when integrating with manufacturing or construction toolchains.

Chapter 64 covers glTF 2.0, the runtime-delivery standard for a *renderable scene*: geometry, materials, and a flat node hierarchy, optimized for GPU upload. glTF has no native concept of "this bracket is part of this assembly, which is part of this product, and here is its supplier part number, IFC classification, or bill-of-materials position" — nodes are a scene graph, not a semantic ownership hierarchy, and glTF's metadata surface is limited to `extras` and a handful of khronos extensions. No single standard combines "browsable visual catalog," "wiki-editable metadata," and "rigorous subcomponent/assembly hierarchy" the way that combination is sometimes assumed to exist as one thing. Instead, the landscape splits by domain, each with a different tradeoff between rigor, browsability, and geometric fidelity. This chapter surveys that landscape and shows where each standard's object model overlaps with, or could feed, a glTF-based Linux rendering pipeline.

## Table of Contents

- [1. Why glTF Doesn't Model Assemblies or Metadata](#1-why-gltf-doesnt-model-assemblies-or-metadata)
- [2. IFC / ISO 16739 — Building Information Modeling as a Rigorous Object+Metadata+Assembly Standard](#2-ifc--iso-16739--building-information-modeling-as-a-rigorous-objectmetadataassembly-standard)
- [3. ISO 10303 (STEP) — The Manufacturing Interchange Format IFC Derives From](#3-iso-10303-step--the-manufacturing-interchange-format-ifc-derives-from)
- [4. Wikimedia Commons 3D and Wikidata — The Literal "Visual Wikipedia" Attempt](#4-wikimedia-commons-3d-and-wikidata--the-literal-visual-wikipedia-attempt)
- [5. ML-Scale Shape Corpora: ShapeNet and Objaverse/Objaverse-XL](#5-ml-scale-shape-corpora-shapenet-and-objaverseobjaverse-xl)
- [6. Digital Heritage: Smithsonian Voyager as a Working Encyclopedia-Style 3D Catalog](#6-digital-heritage-smithsonian-voyager-as-a-working-encyclopedia-style-3d-catalog)
- [7. Electronics Part Catalogs: Octopart, Digi-Key, SnapEDA, and Ultra Librarian](#7-electronics-part-catalogs-octopart-digi-key-snapeda-and-ultra-librarian)
- [8. Metadata Without Geometry: schema.org Product and GS1 GTIN/GDSN](#8-metadata-without-geometry-schemaorg-product-and-gs1-gtingdsn)
- [9. Comparing the Landscape](#9-comparing-the-landscape)
- [10. Integrations](#10-integrations)

---

## 1. Why glTF Doesn't Model Assemblies or Metadata

glTF 2.0's node hierarchy (Chapter 64) exists to describe a *scene graph* — parent-child transform relationships needed to place and animate geometry — not a *product structure*. A glTF node's children are whatever the exporter decided to nest for transform convenience; there is no schema-level distinction between "this child node is a structural subassembly with its own part number" and "this child node is a decorative decal offset by a matrix." Object identity beyond a display `name` string, engineering metadata (material specification, tolerance, supplier, revision), and explicit composition semantics (`assembly A consists of parts B, C, D`) all fall outside the core specification. The `extras` object and vendor/EXT extension mechanism can carry arbitrary JSON, so *some* of this can be bolted on — but nothing enforces a shared vocabulary across tools, which is exactly the problem the standards in this chapter each solve within their own domain.

The domains below split along a consistent axis: **rigor of the metadata/assembly schema** versus **browsability and geometric richness**. The most rigorously specified standard (IFC) has no public browsable catalog; the most literally "Wikipedia-like" effort (Wikimedia Commons 3D) has the least mature metadata-to-geometry linkage.

---

## 2. IFC / ISO 16739 — Building Information Modeling as a Rigorous Object+Metadata+Assembly Standard

The **Industry Foundation Classes (IFC)** schema, standardized as [ISO 16739-1:2024](https://www.iso.org/standard/84123.html) ("Industry Foundation Classes (IFC) for data sharing in the construction and facility management industries — Part 1: Data schema") and maintained by [buildingSMART International](https://technical.buildingsmart.org/standards/ifc/), is the closest thing the built-environment world has to "objects + metadata + assemblies" as a single, formally registered standard. IFC is an EXPRESS-language entity-relationship schema — the same modeling language STEP uses (§3) — serialized to the STEP physical file format (`ISO 10303-21`), and separately to IFC-XML and, more recently, IFC-JSON and an RDF/OWL representation for linked-data use. The current edition, informally called IFC4.3, added infrastructure domains (rail, road, port, bridge) to what had previously been a buildings-only schema, and became the ISO 16739-1:2024 standard text. [Source](https://www.iso.org/standard/84123.html)

IFC's object model separates three concerns cleanly, which is the structural feature that makes it a genuine "objects + metadata + subcomponents" standard rather than just a geometry format:

- **`IfcObjectDefinition`** (and its subtype `IfcObject`) is the base class for anything the schema treats as a distinct real-world thing — walls, beams, doors, spaces, and also non-physical objects like tasks or organizational actors. Physical building elements (`IfcWall`, `IfcBeam`, `IfcDoor`, `IfcColumn`, etc.) subtype `IfcElement`, itself a subtype of `IfcObject`, and carry an explicit geometric representation (`IfcProductRepresentation`) separate from their identity.
- **`IfcPropertyDefinition`** is the extensible metadata layer. `IfcPropertySet` groups arbitrary named `IfcProperty` values (single value, enumerated, bounded, or tabular) and attaches to any object via the `IfcRelDefinesByProperties` relationship — this is the mechanism by which domain-specific metadata (fire rating, thermal transmittance, supplier, warranty date) attaches to a wall or door without modifying the core schema, analogous in spirit to glTF's `extras` but with a registered, versioned vocabulary (`Pset_WallCommon`, `Pset_DoorCommon`, etc.) maintained by buildingSMART rather than an ad hoc per-exporter convention.
- **`IfcRelationship`** subtypes express composition explicitly as first-class schema entities rather than implicit tree nesting. `IfcRelAggregates` ties components into assemblies (a wall assembly composed of layers; a building composed of storeys composed of spaces) — the direct analog of the "subcomponent metadata" requirement — while `IfcRelConnectsElements` expresses physical connectivity (a beam connected to a column) independent of the aggregation hierarchy, letting the same object participate in both a whole/part tree and a separate connectivity graph simultaneously.

```
-- Simplified EXPRESS excerpt illustrating the IfcRelAggregates pattern
-- (schema text is illustrative of the entity-relationship structure, not a verbatim
--  extract of the ISO 16739-1:2024 EXPRESS long form)
ENTITY IfcRelAggregates
  SUBTYPE OF (IfcRelDecomposes);
    RelatingObject : IfcObjectDefinition;   -- the whole (e.g. an IfcElementAssembly)
    RelatedObjects : SET [1:?] OF IfcObjectDefinition;  -- the parts
END_ENTITY;
```

IFC is the "real deal" for objects + metadata + assemblies **within its domain**: it is scoped to the built environment (buildings and, since IFC4.3, civil infrastructure), not general consumer or mechanical products, and there is no public, Wikipedia-style browsable index of IFC models — IFC files circulate as project deliverables between architecture, engineering, and construction (AEC) software (Revit, ArchiCAD, Bonsai/BlenderBIM, IfcOpenShell-based tooling), not as a searchable public catalog. Chapter 42's Blender coverage is relevant here: [Bonsai](https://bonsaibim.org/) (formerly BlenderBIM) is an open-source Blender add-on built on [IfcOpenShell](https://ifcopenshell.org/) that makes Blender a full IFC authoring and viewing environment on Linux, geometry and property sets included.

---

## 3. ISO 10303 (STEP) — The Manufacturing Interchange Format IFC Derives From

**STEP**, formally [ISO 10303](https://www.iso.org/standard/66654.html) ("Industrial automation systems and integration — Product data representation and exchange"), is the broader manufacturing equivalent IFC's schema language and file format were built on. Where IFC is one purpose-built EXPRESS schema for one industry, STEP is a family of "Application Protocols" (APs) — each a separate EXPRESS schema targeting a specific industrial use case — sharing the same EXPRESS modeling language and `ISO 10303-21` physical-file (`.stp`/`.step`) encoding. The dominant AP for mechanical CAD interchange today is **AP242** ([ISO 10303-242](https://www.iso.org/standard/66654.html)), which unified and superseded the earlier AP203 (configuration-controlled 3D designs) and AP214 (automotive core data) application protocols, adding model-based definition (PMI — product manufacturing information, i.e. GD&T annotations bound directly to geometry) as a core capability. [Source](https://www.prostep.org/en/medialibrary/fact-sheets/iso-10303-242-step-ap242)

STEP's assembly model is the mechanical-CAD counterpart to IFC's `IfcRelAggregates`: a **Next Assembly Usage Occurrence (NAUO)** entity ties a `product_definition` (a part or subassembly) into a parent product's structure, carrying its own placement transform — the same "product composed of positioned sub-products" pattern IFC expresses for buildings, generalized to arbitrary mechanical assemblies and bills of materials. AP242's breakdown-structure extensions additionally support functional, physical, system, and zonal breakdowns of the same product — multiple simultaneous decomposition views of one assembly, addressing the same "more than one kind of subcomponent relationship" need that IFC handles by keeping `IfcRelAggregates` (composition) and `IfcRelConnectsElements` (connectivity) as separate relationship types.

STEP is explicitly an **interchange format**, not a browsable catalog: CAD tools (Siemens NX, CATIA, SolidWorks, and the open-source [Open CASCADE Technology](https://dev.opencascade.org/) kernel covered in Chapter 176) read and write `.step` files to move assemblies and their metadata between systems, but there is no public STEP repository analogous to what the question envisions for a "visual Wikipedia." Chapter 176's coverage of OCCT's `STEPControl` reader/writer is the concrete Linux-stack touchpoint for everything described in this section.

---

## 4. Wikimedia Commons 3D and Wikidata — The Literal "Visual Wikipedia" Attempt

Wikimedia's own effort is the closest literal match to "a visual Wikipedia of physical objects," and it is genuinely two separate, loosely coupled systems working together.

**Wikimedia Commons** has accepted `.stl` file uploads since 2018 via the [MediaWiki `3D` extension](https://www.mediawiki.org/wiki/Extension:3D), rendering an interactive preview via a WebGL viewer directly in the file page. STL is a bare-geometry format — no material, texture, color, or hierarchy — so a Commons 3D file page today is close to Wikipedia's photo-library model applied to raw mesh geometry: browsable, freely licensed, and wiki-editable at the metadata-page level, but without the rich per-material, per-node metadata a glTF or STEP file can carry. As of August 2025 a community- and Wikimedia-Foundation-developed prototype ([mediawiki3d.org](https://mediawiki3d.org/index.php/Main_Page)) extended this to `.glb` (glTF 2.0 binary) uploads, adding textures, materials, and animation support and importing a large batch of freely licensed Sketchfab models as a seed corpus — this closes much of the "untextured" gap the STL-only baseline had, though as a prototype rather than a shipped default Commons feature it should be treated as a direction, not yet a stable guarantee. [Source](https://commons.wikimedia.org/wiki/Commons:Textured_3D)

**Wikidata**, the structured-data sibling project, is where the "subcomponent metadata" half of the picture lives, independent of whichever 3D file format a given item links to. [Property P527 ("has part(s)")](https://www.wikidata.org/wiki/Property:P527) and its inverse [Property P361 ("part of")](https://www.wikidata.org/wiki/Property:P361) let any Wikidata item declare a whole/part relationship to any other item — a bicycle item can declare `has part(s): wheel`, `has part(s): frame`, `has part(s): derailleur`, each a separate Wikidata item with its own statements, and each of those can in turn point to a Commons 3D file via the `3D model` property. This is a genuine, generic, crowd-editable assembly graph in the same conceptual family as IFC's `IfcRelAggregates` or STEP's NAUO — but with none of the schema-level rigor: nothing enforces that a P527 claim reflects a real physical containment relationship rather than a categorical one (Wikidata also uses `part of` for things like "administrative territorial entity is part of a country"), and there is no requirement that a linked 3D model's own mesh hierarchy correspond to the P527/P361 graph at all. A query joining Commons 3D files to their Wikidata parent items and P527/P361 subcomponent graph illustrates the intended shape of the combination:

```sparql
# Wikidata Query Service — find items with a linked 3D model and their declared parts
SELECT ?item ?itemLabel ?model3d ?part ?partLabel WHERE {
  ?item wdt:P1267 ?model3d .          # has an associated 3D model (Commons file)
  OPTIONAL { ?item wdt:P527 ?part . } # has part(s)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

The result is a real but immature system: broader in scope than IFC or STEP (any physical object, not one industry), genuinely wiki-editable and publicly browsable in the way the question's premise assumes, but structurally looser — a crowd-sourced knowledge graph loosely linked to a growing but still STL/glTF-mixed geometry corpus, rather than one schema enforcing that geometry, metadata, and assembly structure agree with each other.

---

## 5. ML-Scale Shape Corpora: ShapeNet and Objaverse/Objaverse-XL

A second cluster of efforts approaches the same "taxonomy-organized visual catalog with subcomponents" shape from the machine-learning research side — built for training data, not public browsing, but structurally relevant because of how they organize objects.

**ShapeNet**, introduced in [Chang et al., "ShapeNet: An Information-Rich 3D Model Repository" (2015)](https://arxiv.org/abs/1512.03012), organizes several million 3D CAD models by [WordNet](https://wordnet.princeton.edu/) synsets — the same lexical-taxonomy structure ImageNet uses for image categories — giving it a genuine, if coarse, semantic taxonomy rather than an arbitrary folder structure. **ShapeNetCore**, a cleaned subset of roughly 51,000 models across 55 common categories with manually verified alignment, is the version most commonly used in research; **ShapeNetPart** extends a subset of ShapeNetCore with per-vertex part-segmentation labels (roughly 50 labeled part categories across 16 object classes), making it the closest thing in this cluster to "taxonomy-organized catalog with subcomponents" — though the "subcomponents" are semantic segmentation labels on a single mesh, not a compositional assembly graph of separately identifiable, separately transformable parts the way an IFC or STEP assembly is. [Source](https://shapenet.org/about)

**Objaverse** and its successor **Objaverse-XL**, from the Allen Institute for AI (AI2) in collaboration with several university and industry labs, scale the same idea by orders of magnitude rather than curating a taxonomy: [Objaverse-XL: A Universe of 10M+ 3D Objects (2023)](https://arxiv.org/abs/2307.05663) aggregates and deduplicates over ten million 3D objects from heterogeneous sources — manually modeled assets, photogrammetry scans, and professional scans of cultural artifacts — with per-object licensing that varies by source rather than a single blanket license, and the dataset as a whole distributed under an [ODC-By v1.0](https://github.com/allenai/objaverse-xl) attribution license. Objaverse-XL deliberately trades curation and taxonomic organization for scale: it is explicitly positioned as training data for generative 3D and multimodal models, not a browsable encyclopedia, and has no equivalent to ShapeNet's WordNet-grounded category structure or IFC/STEP's formal metadata schema.

---

## 6. Digital Heritage: Smithsonian Voyager as a Working Encyclopedia-Style 3D Catalog

The [Smithsonian Institution's Digitization Program Office](https://3d.si.edu/open-source-resources) publishes an open-source toolchain — **Voyager** — that is arguably the most concretely "encyclopedia-like" working example among all the options in this chapter, because it combines public browsability, rich curatorial metadata, and glTF-based geometry in one shipped system rather than as separate, loosely linked efforts. [Source](https://smithsonian.github.io/dpo-voyager/)

Voyager is a suite of open-source tools rather than a single application: **Voyager Story** is the curatorial authoring tool used to attach annotations, articles, and guided tours to a 3D model; **Voyager Explorer** is the embeddable web viewer component that renders the result; and a companion **Cook** service handles server-side 3D data processing (mesh decimation, UV unwrapping, texture baking) feeding the pipeline. The published, viewer-consumed asset format is glTF 2.0 (`.glb`/`.gltf`) or a Voyager-specific scene package wrapping glTF geometry with the annotation/article/tour metadata layered on top — meaning Voyager's metadata model sits *on top of* the same runtime format Chapter 64 covers, rather than replacing it, and is the most direct example in this chapter of a non-Khronos metadata layer built to interoperate with glTF instead of competing with it. Individual museum objects — a fossil, a historic aircraft, a sculpture — are published with curatorial metadata (provenance, materials, condition, interpretive text) attached directly to specific regions or components of the model via Voyager's annotation system, giving a genuinely "object + wiki-style narrative metadata + visual geometry" result at the level of an individual published artifact, even though Voyager itself has no global cross-object taxonomy or subcomponent-assembly schema comparable to IFC or STEP.

---

## 7. Electronics Part Catalogs: Octopart, Digi-Key, SnapEDA, and Ultra Librarian

Electronics distribution is the domain-specific example where "part metadata + CAD subcomponent model" is most thoroughly commercialized, though split across separate services rather than unified in one standard. **Octopart** aggregates distributor pricing, stock, and datasheet metadata across manufacturers and distributors, and integrates with **SnapEDA** (rebranded **SnapMagic Search**) for the CAD side of the same part records — schematic symbol, PCB footprint, and 3D step/CAD model — so that a single part search resolves to both commercial metadata (price, stock, lifecycle status) and design-ready CAD subcomponent geometry. [Digi-Key](https://www.digikey.com/) separately offers the same symbol/footprint/3D-model triad through a first-party integration with **Ultra Librarian**, covering roughly a million in-stock parts across more than twenty CAD tool export formats (KiCad, Altium, Eagle, OrCAD, and others). [Source](https://www.digikey.com/en/news/press-releases/2017/apr/digi-key-offers-free-ultra-librarian-symbols)

This cluster is worth including precisely because it demonstrates the "objects + subcomponent CAD models + structured metadata" combination working at commercial scale and public browsability — anyone can search Octopart or Digi-Key's catalog and download a part's 3D model for free — but the "assembly" relationship it captures is shallow relative to IFC or STEP: a part record links to *its own* CAD geometry, not to a formal composition hierarchy showing how that part nests into a board, enclosure, or product. KiCad (an open-source EDA tool that runs natively on Linux) is the most direct consumer of these libraries in the FOSS graphics/CAD ecosystem, importing SnapEDA/Ultra Librarian symbol-footprint-3D triples directly into a schematic and PCB layout project.

---

## 8. Metadata Without Geometry: schema.org Product and GS1 GTIN/GDSN

The final cluster drops geometry entirely and solves only the metadata half of the problem, at internet e-commerce scale — worth including because it is the standard actually driving most of the world's "what is this physical product" structured data, even without a 3D model attached.

[schema.org's `Product` vocabulary](https://schema.org/gtin13) provides a JSON-LD/microdata schema web publishers embed to describe an offered product — name, brand, `gtin8`/`gtin13`/`gtin14` identifiers, price, availability — consumed primarily by search engines for rich-result display:

```json
{
  "@context": "https://schema.org/",
  "@type": "Product",
  "name": "Example Widget",
  "gtin13": "0012345678905",
  "brand": { "@type": "Brand", "name": "Example Corp" },
  "hasPart": { "@type": "Product", "name": "Replacement Cartridge" }
}
```

The `gtin*` identifiers trace back to [GS1](https://www.gs1.org/), the standards body governing the barcode-encoded Global Trade Item Number, and to the [Global Data Synchronization Network (GDSN)](https://www.gs1.org/services/gdsn/global-data-model) — the interconnected data-pool infrastructure retailers and manufacturers use to exchange authoritative product master data (dimensions, packaging hierarchy, certifications) keyed by GTIN. The [GS1 Web Vocabulary](https://www.gs1.org/gs1-web-vocabulary) extends schema.org's `Product` type with the fuller GS1/GDSN attribute set for organizations that need more than schema.org's baseline covers. None of this layer carries or requires 3D geometry at all — it is the metadata-only end of the spectrum this chapter surveys, included because it is the standard e-commerce systems actually use at scale, and because `schema.org`'s `hasPart`/`isPartOf` properties are the same whole/part pattern seen in IFC's `IfcRelAggregates`, STEP's NAUO, and Wikidata's P527/P361 — the composition-relationship idea recurs in every domain in this chapter, independently reinvented at each layer of rigor.

---

## 9. Comparing the Landscape

| Standard / System | Domain | Geometry | Metadata rigor | Subcomponent model | Publicly browsable |
|---|---|---|---|---|---|
| IFC / ISO 16739 | Construction, AEC | Full 3D (via STEP/IFC-XML) | High — registered `Pset_*` vocabularies | Explicit (`IfcRelAggregates`, `IfcRelConnectsElements`) | No (project files, not a public catalog) |
| STEP / ISO 10303 | Manufacturing, mechanical CAD | Full 3D + PMI (AP242) | High — AP-specific EXPRESS schemas | Explicit (NAUO, multi-view breakdown structures) | No (interchange format) |
| Wikimedia Commons 3D + Wikidata | General / crowd-sourced | STL (geometry-only) or glTF (prototype, textured) | Low-to-medium — free-form claims, no enforced domain schema | Loose (P527/P361, not geometry-linked) | Yes |
| ShapeNet / ShapeNetPart | ML research | Mesh + part segmentation labels | Medium — WordNet taxonomy | Segmentation labels, not a compositional graph | Yes (research download, not a browsing UI) |
| Objaverse-XL | ML research | Mixed-quality mesh, 10M+ objects | Low — aggregated, per-source licensing | None | Yes (bulk download, not a browsing UI) |
| Smithsonian Voyager | Cultural heritage | glTF 2.0 | High per-object (curatorial annotation) | Annotation-level, not a formal assembly schema | Yes |
| Octopart / Digi-Key / SnapEDA / Ultra Librarian | Electronics distribution | CAD symbol/footprint/3D model | High per-part (commercial metadata) | Shallow (part-level, not board/product hierarchy) | Yes |
| schema.org Product + GS1 GTIN/GDSN | E-commerce | None | High (GDSN) / medium (schema.org baseline) | `hasPart`/`isPartOf`, metadata-only | Yes (as embedded page markup) |

No row satisfies every column — which is the structural answer to why no single dominant standard combines "visual," "wiki," and "subcomponent metadata" the way that combination is sometimes assumed to already exist as one thing.

---

## 10. Integrations

- **Chapter 64** (glTF 2.0 — The 3D Asset Pipeline Standard) — the runtime geometry format that Voyager (§6) and the Wikimedia Commons 3D prototype (§4) build their metadata layer on top of, and the target format IFC/STEP tooling (§2–§3) typically converts to for real-time visualization.
- **Chapter 176** (Open CASCADE Technology and CAD Kernels) — the open-source geometry kernel whose `STEPControl` reader/writer is the concrete Linux implementation of the STEP interchange described in §3.
- **Chapter 42** (Blender GPU — Cycles and EEVEE) — Bonsai/BlenderBIM and IfcOpenShell (§2) make Blender a Linux-native IFC authoring and viewing environment.
- **Chapter 244** (Blender AI and MCP) — generative and AI-assisted 3D asset pipelines increasingly draw training and reference data from the ML-scale corpora in §5.
- **Chapter 212** (Python 3D ML Libraries — Open3D, PyTorch3D, and Kaolin) — the tooling most commonly used to load and process ShapeNet and Objaverse/Objaverse-XL data (§5) for research and training workloads.
- **Chapter 115** (NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting) — photogrammetry-derived object scans of the kind Objaverse-XL aggregates (§5) are frequently produced by the same NeRF/splatting pipelines this chapter covers.
- **Chapter 241** (GIMP, Krita, and darktable) — texture-authoring tooling relevant to preparing material maps for glTF/Voyager publication pipelines (§6).
