+++
title = "Work"
description = "Experience, education, and what I'm available for."
path = "work"
template = "page.html"
aliases = ["/cv", "/hire"]
+++

<p class="cv-contact">
{{ email() }} ·
<a href="https://linkedin.com/in/luizmug">linkedin.com/in/luizmug</a> ·
<a href="/cv.pdf">CV PDF</a>
</p>

I'm a programmer and mathematician based in São Paulo (UTC−3), working mostly on low-level systems,
computer graphics, and applied mathematics.

{% available_for_hire() %}
## Available for contract and consulting work

I take on a small number of contract projects per year in:

- **Low-level systems**: performance-aware engineering, CPU and GPU optimisation, threading, SIMD.
- **Computer graphics**: rendering, geometry processing, physics simulations.
- **Applied mathematics**: engineering research & development requiring deeper mathematics and
  physics knowledge.

I prefer scoped engagements with concrete deliverables --- a few days of focused consulting up to
multi-month development. I do not take work on staff augmentation, AI, front-end web-development, or
SaaS-CRUD.

To start a conversation, email <{{ config.extra.email }}>. A short note describing the problem,
expected scope, and timeline is enough --- no boilerplate needed. Response within a day or two.
{% end %}

## Experience

{% cvitem(
    primary="Founder & Lead Developer",
    secondary="Presheaf Studio",
    date="April 2024 – Present",
    location="São Paulo, Brazil") %}
- Designing a native cross-platform application for professional digital sculpting based on my
  research on polygonal mesh retopology.
- Creating novel SIMD-optimised algorithms with multi-threading capabilities for mesh editing.
{% end %}

{% cvitem(
    primary="Software Development Engineer (SDE)",
    secondary="Amazon",
    date="February 2025 – Present",
    location="São Paulo, Brazil") %}
- Spearheaded an invoice processing system for the Indian government handling 1 million updates per
  day.
- Led the implementation of the new transitive token security protocol for spec-compliancy of our
  global invoicing services.
- Ported the legacy testing framework of Amazon's world-wide store-front website for all of our
  invoice solutions.
- Worked on-call for our invoice pipeline with direct contact with both clients and third-party
  suppliers.
{% end %}

{% cvitem(
    primary="Fullstack Python Contractor",
    secondary="NDA startup",
    date="June 2023 – March 2024",
    location="São Paulo, Brazil") %}
- Built a full web app for medical ultrasound courses hosted on Google Cloud.
- Used FastAPI and HTMX with Jinja2 templates and Tailwind to build a fluid user experience.
- Implemented user authentication using SQLite and a distributed asset versioning system using S3.
{% end %}

## Education

{% cvitem(
    primary="University of São Paulo",
    secondary="Master's in Computer Graphics",
    date="Aug. 2024 – Present",
    location="São Paulo, Brazil") %}
I research novel techniques for geometry processing of 3D meshes leveraging ideas from the growing
field of Discrete Differential Geometry. As part of my work, I'm developing algorithms for better
re-topology workflows.
{% end %}

{% cvitem(
    primary="University of São Paulo",
    secondary="BSc in Molecular Sciences",
    gpa="9.2/10",
    date="Jul. 2020 – Aug. 2024",
    location="São Paulo, Brazil") %}
Interdisciplinary scientific research in Computer Science and Mathematics, originating in
combinatorial models of [Homotopy Theory](https://ncatlab.org/nlab/show/homotopy+theory), my studies
evolved into the discretization of continuous surfaces and its applications to Computer
Graphics. This led me into researching differential methods for the geometry processing of polygonal
meshes. I've documented all of my studies in a [comprehensive compendium of Mathematics and Computer Science](/deep-dive).

Relevant course-work: Algorithms, Data Structures, Computer Graphics, Differential Geometry,
Algebraic Topology.
{% end %}

## Technical Skills

**Programming languages:** C++, C, Rust, Python, Lua, Java, Typescript, Javascript.

**Libraries:** Modern C++ STL, C11 stdlib, WinAPI, Linux ABI, Vulkan, OpenGL.
