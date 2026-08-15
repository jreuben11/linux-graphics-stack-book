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
- [7. Commercial 3D Marketplaces: Sketchfab as the Mass-Market Baseline](#7-commercial-3d-marketplaces-sketchfab-as-the-mass-market-baseline)
- [8. Electronics Part Catalogs: Octopart, Digi-Key, SnapEDA, and Ultra Librarian](#8-electronics-part-catalogs-octopart-digi-key-snapeda-and-ultra-librarian)
- [9. Metadata Without Geometry: schema.org Product and GS1 GTIN/GDSN](#9-metadata-without-geometry-schemaorg-product-and-gs1-gtingdsn)
- [10. Comparing the Landscape](#10-comparing-the-landscape)
- [11. Roadmap](#11-roadmap)
- [12. Integrations](#12-integrations)

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

**Data format: the .ifc STEP physical file.** IFC's default serialization is the same `ISO-10303-21` "STEP physical file" (SPF) syntax §3 covers for mechanical CAD: a flat, comma-separated entity-instance list where every object is a numbered `#id = ENTITY_NAME(...)` line, and every relationship is expressed by cross-referencing another instance's number rather than by nesting. A real excerpt from a public IFC4 coordination-view test model shows the pattern directly — note `#41`'s `IFCRELAGGREGATES` referencing `#38` and `#34` by number, the same composition mechanism sketched in the EXPRESS excerpt above, now as it actually appears on disk:

```
ISO-10303-21;
HEADER;
FILE_DESCRIPTION (
        ('ViewDefinition [CoordinationView, QuantityTakeOffAddOnView]'),
        '2;1');
FILE_NAME (
        'example.ifc', '2012-09-24T14:40:06', ('Architect'),
        ('Building Designer Office'), 'IFC Engine DLL version 1.03 beta',
        'IFC Engine DLL version 1.03 beta', 'The authorising person');
FILE_SCHEMA (('IFC4RC4'));
ENDSEC;
DATA;
#2  = IFCOWNERHISTORY(#3, #6, $, .NOTDEFINED., $, $, $, 1348486806);
#41 = IFCRELAGGREGATES('0iL9EJKq95bRo5gB3il1c2', #2, 'BuildingContainer',
        'BuildingContainer for BuildigStories', #34, (#38));
#45 = IFCPROPERTYSET('3tHNUXbZH5vxePn_rLyQFg', #2, 'Pset_WallCommon', $,
        (#46, #47, #48, #49, #50, #51, #52, #53, #54, #55));
#46 = IFCPROPERTYSINGLEVALUE('Reference', 'Reference', IFCIDENTIFIER(''), $);
#60 = IFCWALL('3IclONJQ5D5gm$TM3V7U1j', #2, 'Outer Wall Back',
        'Description of Wall', $, #61, #65, $, .STANDARD.);
ENDSEC;
END-ISO-10303-21;
```
[Source](https://github.com/opensourceBIM/TestFiles/blob/master/TestData/data/ifc4.ifc)

**API: IfcOpenShell.** [IfcOpenShell](https://ifcopenshell.org/) is the open-source C++/Python library underlying Bonsai and most Linux IFC tooling; its Python API opens a model, queries entities by IFC type, and resolves attached property sets in a few lines:

```python
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open('model.ifc')

for wall_type in model.by_type("IfcWallType"):
    print(wall_type.Name)

wall = model.by_type("IfcWall")[0]
psets = ifcopenshell.util.element.get_psets(wall)
# {"Pset_WallCommon": {"id": 123, "FireRating": "2HR", ...}}
```
[Source](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html)

IFC is the "real deal" for objects + metadata + assemblies **within its domain**: it is scoped to the built environment (buildings and, since IFC4.3, civil infrastructure), not general consumer or mechanical products, and there is no public, Wikipedia-style browsable index of IFC models — IFC files circulate as project deliverables between architecture, engineering, and construction (AEC) software (Revit, ArchiCAD, Bonsai/BlenderBIM, IfcOpenShell-based tooling), not as a searchable public catalog. Chapter 42's Blender coverage is relevant here: [Bonsai](https://bonsaibim.org/) (formerly BlenderBIM) is an open-source Blender add-on built on [IfcOpenShell](https://ifcopenshell.org/) that makes Blender a full IFC authoring and viewing environment on Linux, geometry and property sets included.

---

## 3. ISO 10303 (STEP) — The Manufacturing Interchange Format IFC Derives From

**STEP**, formally [ISO 10303](https://www.iso.org/standard/66654.html) ("Industrial automation systems and integration — Product data representation and exchange"), is the broader manufacturing equivalent IFC's schema language and file format were built on. Where IFC is one purpose-built EXPRESS schema for one industry, STEP is a family of "Application Protocols" (APs) — each a separate EXPRESS schema targeting a specific industrial use case — sharing the same EXPRESS modeling language and `ISO 10303-21` physical-file (`.stp`/`.step`) encoding. The dominant AP for mechanical CAD interchange today is **AP242** ([ISO 10303-242](https://www.iso.org/standard/66654.html)), which unified and superseded the earlier AP203 (configuration-controlled 3D designs) and AP214 (automotive core data) application protocols, adding model-based definition (PMI — product manufacturing information, i.e. GD&T annotations bound directly to geometry) as a core capability. [Source](https://www.prostep.org/en/medialibrary/fact-sheets/iso-10303-242-step-ap242)

STEP's assembly model is the mechanical-CAD counterpart to IFC's `IfcRelAggregates`: a **Next Assembly Usage Occurrence (NAUO)** entity ties a `product_definition` (a part or subassembly) into a parent product's structure, carrying its own placement transform — the same "product composed of positioned sub-products" pattern IFC expresses for buildings, generalized to arbitrary mechanical assemblies and bills of materials. AP242's breakdown-structure extensions additionally support functional, physical, system, and zonal breakdowns of the same product — multiple simultaneous decomposition views of one assembly, addressing the same "more than one kind of subcomponent relationship" need that IFC handles by keeping `IfcRelAggregates` (composition) and `IfcRelConnectsElements` (connectivity) as separate relationship types.

**Data format: the .stp physical file.** Like IFC (§2), STEP's file encoding is `ISO-10303-21` SPF syntax — the two schemas share a physical-file format because IFC's authors deliberately reused STEP's existing serialization rather than inventing a new one. A minimal real excerpt (AP214, the automotive-core-data protocol AP242 superseded — no equivalently minimal AP242-labeled sample was located, though AP242 files use the identical SPF structure with a longer `FILE_SCHEMA` identifier, e.g. `AP242_MANAGED_MODEL_BASED_3D_ENGINEERING_MIM_LF`; *Note: needs verification* for that exact current schema identifier string):

```
ISO-10303-21;
HEADER;
FILE_DESCRIPTION(
/* description */ ('A minimal AP214 example with a single part'),
/* implementation_level */ '2;1');
FILE_NAME(
/* name */ 'demo',
/* time_stamp */ '2003-12-27T11:57:53',
/* author */ ('Lothar Klein'),
/* organization */ ('LKSoft'),
/* preprocessor_version */ ' ',
/* originating_system */ 'IDA-STEP',
/* authorization */ ' ');
FILE_SCHEMA (('AUTOMOTIVE_DESIGN { 1 0 10303 214 2 1 1}'));
ENDSEC;
DATA;
#14=PRODUCT_DEFINITION('0',$,#15,#11);
#15=PRODUCT_DEFINITION_FORMATION('1',$,#16);
#16=PRODUCT('A0001','Test Part 1','',(#18));
#18=PRODUCT_CONTEXT('',#12,'');
ENDSEC;
END-ISO-10303-21;
```
[Source](https://en.wikipedia.org/wiki/ISO_10303-21)

**API: OpenCASCADE `STEPControl`.** OCCT (Chapter 176) reads and writes STEP through a dedicated reader/writer pair that transfers between a `.stp` file and OCCT's own `TopoDS_Shape` B-rep representation:

```cpp
#include <STEPControl_Reader.hxx>

STEPControl_Reader reader;
reader.ReadFile("MyFile.stp");
reader.TransferRoots();
TopoDS_Shape result = reader.OneShape();
```

```cpp
#include <STEPControl_Writer.hxx>

STEPControl_Writer writer;
writer.Transfer(shape, STEPControl_ManifoldSolidBrep);
writer.Write("filename.stp");
```
[Source](https://dev.opencascade.org/doc/overview/html/occt_user_guides__step.html)

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

**API: the `objaverse` Python package.** Both generations are consumed as a Python library rather than a browsing UI, reinforcing that this cluster targets ML pipelines, not catalog visitors. The original Objaverse 1.0 API resolves UIDs to local downloaded `.glb` files:

```python
import objaverse

uids = objaverse.load_uids()
annotations = objaverse.load_annotations(uids[:10])

objects = objaverse.load_objects(uids=uids[:10], download_processes=4)
# -> {uid: local_glb_path, ...}
```
[Source](https://objaverse.allenai.org/docs/objaverse-1.0/)

Objaverse-XL's own package exposes a broader, multi-source-aware surface (GitHub, Thingiverse, and other origins beyond Sketchfab) rather than this exact call shape — treat the snippet above as illustrative of the 1.0 API specifically. [Source](https://github.com/allenai/objaverse-xl)

---

## 6. Digital Heritage: Smithsonian Voyager as a Working Encyclopedia-Style 3D Catalog

The [Smithsonian Institution's Digitization Program Office](https://3d.si.edu/open-source-resources) publishes an open-source toolchain — **Voyager** — that is arguably the most concretely "encyclopedia-like" working example among all the options in this chapter, because it combines public browsability, rich curatorial metadata, and glTF-based geometry in one shipped system rather than as separate, loosely linked efforts. [Source](https://smithsonian.github.io/dpo-voyager/)

Voyager is a suite of open-source tools rather than a single application: **Voyager Story** is the curatorial authoring tool used to attach annotations, articles, and guided tours to a 3D model; **Voyager Explorer** is the embeddable web viewer component that renders the result; and a companion **Cook** service handles server-side 3D data processing (mesh decimation, UV unwrapping, texture baking) feeding the pipeline. The published, viewer-consumed asset format is glTF 2.0 (`.glb`/`.gltf`) or a Voyager-specific scene package wrapping glTF geometry with the annotation/article/tour metadata layered on top — meaning Voyager's metadata model sits *on top of* the same runtime format Chapter 64 covers, rather than replacing it, and is the most direct example in this chapter of a non-Khronos metadata layer built to interoperate with glTF instead of competing with it. Individual museum objects — a fossil, a historic aircraft, a sculpture — are published with curatorial metadata (provenance, materials, condition, interpretive text) attached directly to specific regions or components of the model via Voyager's annotation system, giving a genuinely "object + wiki-style narrative metadata + visual geometry" result at the level of an individual published artifact, even though Voyager itself has no global cross-object taxonomy or subcomponent-assembly schema comparable to IFC or STEP.

---

## 7. Commercial 3D Marketplaces: Sketchfab as the Mass-Market Baseline

Every system covered so far trades off rigor and browsability at a relatively small scale — a few thousand curated Voyager artifacts, a few hundred thousand STL/glTF Commons uploads. **Sketchfab** is the opposite extreme on the browsability axis: a general-purpose commercial 3D marketplace, founded in Paris in 2012, that had grown to **8M+ hosted models, 25M+ registered users, and 6M+ unique monthly visitors** by the time of writing, dwarfing every other public catalog in this chapter by an order of magnitude or more. [Source](https://sketchfab.com/about) Epic Games acquired Sketchfab in July 2021 and cut its Store commission from 30% to 12%; [Source](https://www.epicgames.com/site/en-US/news/sketchfab-is-now-part-of-epic-games) Epic then sold both Sketchfab and ArtStation to KitBash (the KitBash3D/Greyscalegorilla group) on August 10, 2026, retaining only Fab as its own asset marketplace — a change recent enough at the time of writing that it is worth flagging explicitly rather than assuming "Epic-owned" remains accurate by the time this book is read. [Source](https://www.gamedeveloper.com/business/epics-sells-artstation-and-sketchfab-to-kitbash)

Sketchfab is included in this chapter specifically because it is the **mass-market counterexample** to the rest of the survey: it has by far the best public browsability and the largest corpus, but close to the weakest metadata rigor and no subcomponent/assembly model at all — the inverse tradeoff from IFC or STEP.

- **Upload and conversion pipeline.** Sketchfab accepts uploads in 50+ formats (OBJ, FBX, Collada, 3DS, Blender `.blend`, STL, VRML, USD, and glTF itself among them), with direct-export plugins for Blender, Maya, Cinema 4D, SketchUp, SolidWorks, and ZBrush. [Source](https://sketchfab.com/features) glTF/GLB is the platform's own recommended upload format, and Sketchfab converts whatever format is uploaded into glTF/GLB (and USDZ) for its download pipeline — the company has marketed itself as "the largest online repository of glTF files" since adding native glTF upload support in 2016 and a glTF-based Download API in 2018. [Source](https://www.khronos.org/blog/sketchfab-uses-gltf-to-bring-a-search-bar-to-the-world-of-3d) [Source](https://sketchfab.com/features/gltf) Note: needs verification — no primary source describes the specific rendering engine underlying Sketchfab's embedded WebGL viewer (whether Three.js-based or fully in-house); the feature page only advertises "no plugin required" cross-browser support.
- **Metadata model.** The Data API (v3) exposes model-level fields — `uid`, `name`, `description`, `tags`, `categories`, `license`, `faceCount`, `vertexCount`, `isDownloadable`, plus engagement and publication fields (`viewCount`, `likeCount`, `viewerUrl`, `embedUrl`, `thumbnails`, `user`) — all scoped to the whole model. A real (trimmed) response for `GET https://api.sketchfab.com/v3/models/{uid}`:
  ```json
  {
    "uid": "70fa0927a36a4bc392f3e13c882d287e",
    "name": "Chairs Pack / Chair Collection",
    "viewerUrl": "https://sketchfab.com/3d-models/none-70fa0927a36a4bc392f3e13c882d287e",
    "faceCount": 194582,
    "isDownloadable": false,
    "license": { "label": "Standard", "slug": "st", "url": "https://sketchfab.com/licenses" },
    "categories": [{ "name": "Furniture & Home", "slug": "furniture-home" }],
    "tags": [{ "name": "office", "slug": "office" }],
    "user": { "username": "studiolab.dev", "displayName": "Studio Lab" }
  }
  ```
  [Source](https://sketchfab.com/developers/data-api/v3) No endpoint exposes per-node, per-mesh, or scene-graph metadata: unlike IFC's `IfcRelAggregates` (§2) or STEP's NAUO (§3), a Sketchfab listing carries exactly one license, one tag set, and one aggregate triangle count for the entire uploaded scene, however many named parts the underlying glTF or FBX hierarchy actually contains. Whatever compositional structure exists is invisible above the file format itself — Sketchfab solves discovery and browsability, not the "subcomponent metadata" half of this chapter's framing at all.
- **Licensing.** The default is Sketchfab's own **Standard license** (commercial use and derivatives permitted with restrictions — no standalone resale, no use in logos/trademarks, attribution required "where technically feasible"), with a separate restrictive **Editorial license** for news/commentary use. [Source](https://sketchfab.com/licenses) For downloadable models, uploaders can instead opt into **Creative Commons** terms (CC BY, with optional NC/ND/SA modifiers), filterable in search; [Source](https://sketchfab.com/blogs/community/an-introduction-to-creative-commons-licenses/) a **CC0/public-domain** option is offered specifically to cultural-heritage institutions publishing 3D scans. [Source](https://sketchfab.com/blogs/community/sketchfab-launches-public-domain-dedication-for-3d-cultural-heritage/) Note: needs verification — no single Sketchfab page was found enumerating the complete current license list in one place; this is assembled from three separate license/blog pages.
- **API surface.** Beyond the Data API, a **Download API** returns glTF/GLB/USDZ download links for authenticated requests, and a separate **Viewer API** plus oEmbed support cover embedding the interactive viewer in third-party pages. [Source](https://sketchfab.com/developers) [Source](https://sketchfab.com/developers/download-api/guidelines) This is the API surface the `ahujasid/blender-mcp` community server's `search_sketchfab_models`/`download_sketchfab_model` tools (Chapter 244, §3) call.
- **Role as an ML training corpus.** The original **Objaverse** dataset (Allen Institute for AI, 2022) was built entirely from Sketchfab: "objects selected for Objaverse have a distributable Creative Commons license and were obtained using Sketchfab's public API," yielding roughly 818K CC-licensed objects. [Source](https://ar5iv.labs.arxiv.org/html/2212.08051) When AI2 scaled this up to **Objaverse-XL** (2023, §5) to more than 10M objects, Sketchfab's contribution stayed essentially fixed at the original ~800K (≈8% of the expanded total) — the tenfold growth came almost entirely from GitHub (~56%) and Thingiverse (~35%) instead. [Source](https://arxiv.org/html/2307.05663) Sketchfab was Objaverse's entire seed corpus and remains a meaningful but no-longer-dominant slice of Objaverse-XL.
- **Embedding and VR.** Models embed via a configurable iframe or oEmbed, and the viewer advertises headset support (HTC Vive, Oculus/Meta Rift, Gear VR, Google Cardboard/Daydream) in its own marketing copy. Note: needs verification — that copy is phrased in legacy WebVR terms; WebVR was deprecated in browsers years ago in favor of WebXR, so the current implementation is very likely WebXR-based even though Sketchfab's public feature page has not updated the terminology. [Source](https://sketchfab.com/features)

Set against the rest of this chapter, Sketchfab is the clearest illustration of the rigor-versus-browsability tradeoff stated in §1: it is the largest and most publicly browsable catalog surveyed here by a wide margin, yet it satisfies none of "explicit subcomponent model" the way IFC, STEP, or even Wikidata's P527/P361 graph (§4) attempt to, and its per-object metadata is closer to a media-sharing site's (title, tags, license) than a museum catalog's (Voyager, §6) or a manufacturing record's (STEP, §3).

---

## 8. Electronics Part Catalogs: Octopart, Digi-Key, SnapEDA, and Ultra Librarian

Electronics distribution is the domain-specific example where "part metadata + CAD subcomponent model" is most thoroughly commercialized, though split across separate services rather than unified in one standard. **Octopart** aggregates distributor pricing, stock, and datasheet metadata across manufacturers and distributors, and integrates with **SnapEDA** (rebranded **SnapMagic Search**) for the CAD side of the same part records — schematic symbol, PCB footprint, and 3D step/CAD model — so that a single part search resolves to both commercial metadata (price, stock, lifecycle status) and design-ready CAD subcomponent geometry. [Digi-Key](https://www.digikey.com/) separately offers the same symbol/footprint/3D-model triad through a first-party integration with **Ultra Librarian**, covering roughly a million in-stock parts across more than twenty CAD tool export formats (KiCad, Altium, Eagle, OrCAD, and others). [Source](https://www.digikey.com/en/news/press-releases/2017/apr/digi-key-offers-free-ultra-librarian-symbols)

**API: Nexar.** Octopart's original REST API has been superseded by **Nexar**, a GraphQL API (operated by Nexar/Altium) that now covers Octopart's supply-chain search alongside Altium 365 design data and Altium's manufacturing (Altimade) data under one schema, authenticated with OAuth2 rather than the old static API key:

```graphql
{
  supSearchMpn(q: "SN74S74N") {
    hits
    results {
      part {
        mpn
        manufacturer { name }
        sellers { company { name } offers { prices { price quantity } } }
      }
    }
  }
}
```
[Source](https://nexar.com/api) [Source](https://support.nexar.com/support/solutions/articles/101000469281-migration-from-octopart-v4-nexar-legacy-api-)

This cluster is worth including precisely because it demonstrates the "objects + subcomponent CAD models + structured metadata" combination working at commercial scale and public browsability — anyone can search Octopart or Digi-Key's catalog and download a part's 3D model for free — but the "assembly" relationship it captures is shallow relative to IFC or STEP: a part record links to *its own* CAD geometry, not to a formal composition hierarchy showing how that part nests into a board, enclosure, or product. KiCad (an open-source EDA tool that runs natively on Linux) is the most direct consumer of these libraries in the FOSS graphics/CAD ecosystem, importing SnapEDA/Ultra Librarian symbol-footprint-3D triples directly into a schematic and PCB layout project.

---

## 9. Metadata Without Geometry: schema.org Product and GS1 GTIN/GDSN

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

The `gtin*` identifiers trace back to [GS1](https://www.gs1.org/), the standards body governing the barcode-encoded Global Trade Item Number, and to the [Global Data Synchronization Network (GDSN)](https://www.gs1.org/services/gdsn/global-data-model) — the interconnected data-pool infrastructure retailers and manufacturers use to exchange authoritative product master data (dimensions, packaging hierarchy, certifications) keyed by GTIN. The [GS1 Web Vocabulary](https://www.gs1.org/gs1-web-vocabulary) extends schema.org's `Product` type with the fuller GS1/GDSN attribute set for organizations that need more than schema.org's baseline covers, using a `gs1:` namespace layered onto the same JSON-LD `@context` mechanism as the schema.org example above — GS1's own SmartSearch implementation guideline gives a real (trimmed) example of a food product extended with GS1-specific ingredient and allergen properties:

```json
{
  "@context": {
    "gs1": "https://ref.gs1.org/voc/",
    "schema": "http://schema.org/",
    "TradeItem": "schema:Product",
    "gtin13": { "@id": "schema:gtin13" },
    "healthClaimDescription": { "@id": "gs1:healthClaimDescription" },
    "allergenStatement": { "@id": "gs1:allergenStatement" },
    "hasAllergenRelatedInformation": { "@id": "gs1:hasAllergenRelatedInformation", "@type": "@id" }
  },
  "@id": "http://id.manufacturer.com/gtin/05011476100885",
  "@type": ["TradeItem"],
  "gtin13": "5011476100885",
  "healthClaimDescription": "8 Vitamins & Iron, Source of Calcium & High in Fibre",
  "hasAllergenRelatedInformation": {
    "@type": "gs1:AllergenRelatedInformation",
    "allergenStatement": "May contain nut traces"
  }
}
```
[Source](https://www.gs1.org/sites/default/files/docs/gs1-smartsearch/GS1_SmartSearch_Implementation_Guideline.pdf) GS1's own 2015 guideline document uses an older `http://gs1.org/voc/` namespace URI; current implementations should resolve the `gs1:` prefix to `https://ref.gs1.org/voc/` per GS1's present-day vocabulary page, shown corrected above. [Source](https://www.gs1.org/gs1-web-vocabulary)

None of this layer carries or requires 3D geometry at all — it is the metadata-only end of the spectrum this chapter surveys, included because it is the standard e-commerce systems actually use at scale, and because `schema.org`'s `hasPart`/`isPartOf` properties are the same whole/part pattern seen in IFC's `IfcRelAggregates`, STEP's NAUO, and Wikidata's P527/P361 — the composition-relationship idea recurs in every domain in this chapter, independently reinvented at each layer of rigor.

---

## 10. Comparing the Landscape

| Standard / System | Domain | Geometry | Metadata rigor | Subcomponent model | Publicly browsable |
|---|---|---|---|---|---|
| IFC / ISO 16739 | Construction, AEC | Full 3D (via STEP/IFC-XML) | High — registered `Pset_*` vocabularies | Explicit (`IfcRelAggregates`, `IfcRelConnectsElements`) | No (project files, not a public catalog) |
| STEP / ISO 10303 | Manufacturing, mechanical CAD | Full 3D + PMI (AP242) | High — AP-specific EXPRESS schemas | Explicit (NAUO, multi-view breakdown structures) | No (interchange format) |
| Wikimedia Commons 3D + Wikidata | General / crowd-sourced | STL (geometry-only) or glTF (prototype, textured) | Low-to-medium — free-form claims, no enforced domain schema | Loose (P527/P361, not geometry-linked) | Yes |
| ShapeNet / ShapeNetPart | ML research | Mesh + part segmentation labels | Medium — WordNet taxonomy | Segmentation labels, not a compositional graph | Yes (research download, not a browsing UI) |
| Objaverse-XL | ML research | Mixed-quality mesh, 10M+ objects | Low — aggregated, per-source licensing | None | Yes (bulk download, not a browsing UI) |
| Smithsonian Voyager | Cultural heritage | glTF 2.0 | High per-object (curatorial annotation) | Annotation-level, not a formal assembly schema | Yes |
| Sketchfab | General / commercial marketplace | glTF 2.0 (converted on upload/download), 50+ source formats | Low — whole-model tags/license/description only | None (no per-node/subcomponent API) | Yes — largest catalog surveyed here (8M+ models) |
| Octopart / Digi-Key / SnapEDA / Ultra Librarian | Electronics distribution | CAD symbol/footprint/3D model | High per-part (commercial metadata) | Shallow (part-level, not board/product hierarchy) | Yes |
| schema.org Product + GS1 GTIN/GDSN | E-commerce | None | High (GDSN) / medium (schema.org baseline) | `hasPart`/`isPartOf`, metadata-only | Yes (as embedded page markup) |

No row satisfies every column — which is the structural answer to why no single dominant standard combines "visual," "wiki," and "subcomponent metadata" the way that combination is sometimes assumed to already exist as one thing.

---

## 11. Roadmap

**Near-term (6–12 months):**
- The [mediawiki3d.org](https://mediawiki3d.org/index.php/Main_Page) textured-glTF prototype (§4) maturing from a community/Wikimedia-Foundation prototype into a stable Commons feature would be the single biggest near-term closer of the "browsable + rich metadata + real geometry" gap the comparison table (§10) shows no current row satisfying — worth revisiting once it graduates from prototype status.
- Fallout from Sketchfab and ArtStation's August 2026 sale from Epic Games to KitBash (§7): whether the Data/Download API, licensing terms, and the existing Objaverse-sourced corpus (§5) stay stable under new ownership is worth tracking given how much downstream tooling — Objaverse itself, Chapter 244's `blender-mcp` Sketchfab tools — depends on API and licensing continuity.
- Nexar's continued migration of Octopart's supply-chain search fully onto its GraphQL API (§8): any Linux EDA tooling (KiCad plugins, scripts) still targeting the legacy Octopart REST endpoints will need to migrate before those endpoints are retired.

**Medium-term (1–3 years):**
- Whether IFC4.3/ISO 16739-1:2024's infrastructure domains (rail, road, port, bridge, §2) see adoption in mainstream AEC tooling comparable to the buildings-only IFC4 schema they extend, or remain a specialist niche relative to buildings.
- Whether AP242's model-based-definition (PMI) capability becomes the default STEP export mode industry-wide, or geometry-only exports in the AP214 style (§3's illustrative excerpt) remain the practical baseline most tools still produce day to day.
- GS1's own reference material and partner implementations fully retiring the older `gs1.org/voc/` namespace URI still visible in some primary GS1 documentation (§9) in favor of the current `ref.gs1.org/voc/` form.

**Long-term:**
- Whether any effort actually closes the structural gap this chapter's comparison table (§10) surfaces — one system combining IFC/STEP-level subcomponent rigor with Wikidata/Sketchfab-level public browsability — remains genuinely open. Nothing surveyed in this chapter is currently positioned to become that system; §1's framing suggests the rigor-versus-browsability split is structural rather than a temporary gap the industry is converging to close.
- ML-scale corpora (Objaverse-XL, §5) growing further on raw aggregation seem likely to keep outpacing curated, taxonomy-organized efforts (ShapeNet) rather than converging with them, absent a shift in incentives from scale back toward curation.

---

## 12. Integrations

- **Chapter 64** (glTF 2.0 — The 3D Asset Pipeline Standard) — the runtime geometry format that Voyager (§6) and the Wikimedia Commons 3D prototype (§4) build their metadata layer on top of, the format Sketchfab converts every upload into for viewing and download (§7), and the target format IFC/STEP tooling (§2–§3) typically converts to for real-time visualization.
- **Chapter 176** (Open CASCADE Technology and CAD Kernels) — the open-source geometry kernel whose `STEPControl` reader/writer is the concrete Linux implementation of the STEP interchange described in §3.
- **Chapter 42** (Blender GPU — Cycles and EEVEE) — Bonsai/BlenderBIM and IfcOpenShell (§2) make Blender a Linux-native IFC authoring and viewing environment.
- **Chapter 244** (Blender AI and MCP) — generative and AI-assisted 3D asset pipelines increasingly draw training and reference data from the ML-scale corpora in §5, and the community MCP server's `search_sketchfab_models`/`download_sketchfab_model` tools (its §3) call the Sketchfab Data/Download API described in §7 of this chapter.
- **Chapter 212** (Python 3D ML Libraries — Open3D, PyTorch3D, and Kaolin) — the tooling most commonly used to load and process ShapeNet and Objaverse/Objaverse-XL data (§5) for research and training workloads.
- **Chapter 115** (NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting) — photogrammetry-derived object scans of the kind Objaverse-XL aggregates (§5) are frequently produced by the same NeRF/splatting pipelines this chapter covers.
- **Chapter 241** (GIMP, Krita, and darktable) — texture-authoring tooling relevant to preparing material maps for glTF/Voyager publication pipelines (§6).
