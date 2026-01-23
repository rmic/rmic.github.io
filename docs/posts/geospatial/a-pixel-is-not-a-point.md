---
date: 2026-01-22
categories:
  - geospatial
---
# Why a pixel is not a geographic point

TL;DR

A pixel is an image-space measurement, not a location on Earth.
Treating it as a point is a category error that silently breaks geospatial reasoning.


## The intuitive assumption

If you work with images, maps, or computer vision, this assumption feels natural:

*“This object is at pixel (x, y), so I know where it is.”*

A slightly more refined version often follows: *“I know the pixel coordinates and the GPS position of the image, so I can convert that pixel into latitude and longitude.”*

This intuition is not stupid. It feels reasonable, especially if you are used to grids, arrays, coordinates and numerical precision.


Pixels have coordinates. Maps have coordinates.
So why shouldn’t one turn into the other?

Because they exist in different worlds. 

## What a pixel actually is

Long story short : A pixel is **not** a point.

It is a small — sometimes *very* small — rectangular area on a camera sensor. During an exposure, that area integrates incoming light from the scene in front of the camera. Nothing in that process encodes depth, direction, or position in the world.

Even before talking about geography, a pixel already has extent, not location.

A pixel in its simplest form is an area over the camera sensor.

At this stage, nothing in the image tells you:
* where on Earth this pixel comes from
* how far or how large the observed surface is
* or in which direction the scene lies

The pixel exists purely in image space.

### Giving pixels a physical size: Ground Sampling Distance

To make pixels feel more concrete, practitioners often introduce Ground Sampling Distance (GSD).

GSD answers a simple question: *"If this image were taken under ideal conditions, how much ground would one pixel roughly cover?"*

Under strong assumptions (e.g. perfectly flat terrain, nadir view, known altitude, no lens distortion, ...) the GSD can be approximated from:
* flight altitude
* camera focal length
* physical pixel size on the sensor

Increase altitude, and each pixel covers more ground. Change sensor or lens, and the same pixel suddenly represents a whole different surface area.

Understanding this is useful. It immediately breaks the illusion that pixels are dimensionless. A pixel can represent centimeters, decimeters, or meters on the ground, depending on context.

But GSD only describes scale — not location. A pixel with a known ground size can still be misplaced by meters.

### Why GSD is not enough

So far, we have seen that GSD tells you how much ground a pixel **might roughly** represent. It still says nothing about which part of the ground that pixel is actually seeing.

To answer that question, we would need:
* a viewing direction
* a camera pose
* a reference frame
* and a model of the terrain

Without these, GSD provides a comforting number — but no geometric truth.

!!! note "Out of scope (for now)"
    GSD is often treated as constant, but in reality it varies across an image, changes with viewing angle, and breaks down completely on sloped terrain. These effects are real and important — and we will come back to them later — but they are not required to understand the core idea here:

    **a pixel does not have an intrinsic position and its apparent size only exists under strong assumptions**

## The hidden assumptions

Turning a pixel into a geographic point does not fail because people are careless.
It fails because it relies on a stack of assumptions that are rarely stated explicitly.

When you click on an image, run a detection model, or export a point from imagery, you are implicitly assuming that all of the following are true — at the same time.

A pixel corresponds to a single ray

The first assumption is that a pixel can be reduced to a single viewing direction.

In reality, a pixel is an area on the sensor. Light reaching that area arrives from a small cone of directions, not from a perfectly defined ray. Camera models usually collapse that cone into a single central ray because it is convenient — and often sufficient — but this is already an approximation.

For most applications, this approximation is harmless.
For precise geometry, it is the first place where uncertainty enters the system.

### That ray intersects the ground at a unique point

Once a pixel is treated as a ray, a second assumption follows naturally: that the ray intersects the ground once, at a well-defined location.

This is only true if the surface is continuous and there are no vertical structures or occlusions.

In real scenes, a single ray can intersect a roof edge, a façade, vegetation, or the ground behind an object.

Choosing which intersection is “the point” is already a modeling decision — even if it is never exposed as such.


### The ground is planar (or at least predictable)

Most pixel-to-ground conversions quietly assume that the ground can be approximated as flat, horizontal, or very smoothly varying.

This assumption is often justified with phrases like “the area is small” or “the terrain is simple”. But small and flat does not mean level, and simple does not mean planar.

Even gentle slopes introduce sometimes significant lateral displacement.
Even minor relief breaks geometric symmetry.

The flatter the world is assumed to be, the more confident the result appears — and the further it can quietly drift from reality.

### The camera pose is perfectly known

Another strong assumption is that the camera’s position and orientation are known with sufficient accuracy.

In practice, pose comes from:
* GPS / GNSS / Baidu / Galileo measurements,
* RTK adjustments,
* inertial sensors,
* time synchronization,
* calibration

Each of these has its own uncertainty, bias, and failure modes. When pose errors exist — and they always do — they directly translate into ground position errors, often amplified by altitude and viewing angle.

The pixel did not move. The geometry did.

### “Altitude” is known — and meaningful

Finally, many workflows rely on altitude as if it were a single, unambiguous value.

But altitude relative to what?
* mean sea level (AMSL) ? Is your current ground at sea level ? 
* takeoff point ? Is there always at that exact same distance between the camera sensor and the ground beneath it ? 
* local terrain ?
* an ellipsoid ?

Different definitions of altitude can differ by meters. Using the wrong one does not produce an error message — it produces a plausible but wrong result.

This is one of the most common ways silent errors enter geospatial pipelines.

### Why this matters

None of these assumptions is absurd.
Most of them are reasonable — individually.

The problem is that they are rarely acknowledged, rarely verified, and almost never combined consciously.

You do not make these assumptions explicitly. The tools make them for you.

And once they are baked into a result, they are very hard to see — let alone undo.

Understanding that these assumptions exist is the first step toward understanding why treating a pixel as a geographic point is not just imprecise, but conceptually wrong.

## From pixel to ground: what must exist

To understand what it would take to turn a pixel into a geographic location, we need to be explicit about what must exist — whether we acknowledge it or not — in every pixel-to-ground conversion.

This is not about software.
It is about geometry.

### A pixel does not map to a point — it maps to a question

A pixel in an image corresponds to an area perceived in a viewing direction from the camera.

That direction depends on the camera’s internal geometry (sensor, lens, distortion) and the pixel’s position on the sensor.

With this information, we still do not have a location — only a direction.
The moment you ask “where is this pixel on the ground?”, you are implicitly asking “Along this direction, where does the world exist?”

Answering that question requires more than an image.

### A surface must be chosen

To turn a viewing direction into a location, the direction must intersect something.

That “something” is a model of the world such as :
* a flat plane,
* a constant altitude,
* a digital elevation model,
* or any other representation of the terrain.

Choosing a surface is not optional. If no surface is provided, one is assumed.
This is where many workflows quietly commit to a world model without ever naming it and without the user knowing which one it is.

### Intersection is a decision, not a fact

Once a surface exists, the viewing direction can be intersected with it.

But that intersection is not a measurement — it is a result of a choice:
* which surface?
* which altitude reference?
* which resolution?
* which smoothing?

Change the surface, and the intersection moves — sometimes by centimeters, sometimes by meters.

Nothing about the pixel changed. Only the assumptions did.

### Reference frames make locations comparable

Even after an intersection is computed, the result is still not a usable geographic location unless it is expressed in a reference frame.

Coordinates only make sense relative to a reference frame, which is usually defined by three elements:

* a **datum** — a mathematical model of the Earth that defines what “zero” means and how positions relate to the planet’s shape (sphere, ellipsoid, reference surface).
* a **projection** — a rule that transforms positions on a curved surface into a flat, two‑dimensional space, inevitably distorting distances, angles, or areas in the process.
* a **convention** — a set of choices about axes, units, orientation, and coordinate order (for example: meters vs degrees, east‑north vs north‑east, origin placement).

Different choices produce different numbers for the same physical location — all of them internally consistent, none of them universally “correct”.

At this point, the pixel has finally become a geographic hypothesis:

“If the world looks like this, and the camera was there, then the pixel might correspond to this location.”

### Why this matters

Every pixel-to-ground conversion performs these steps, explicitly or implicitly:
1.	derive a viewing direction
2.	choose a surface
3.	compute an intersection
4.	express the result in a reference frame

When these steps are hidden, results look clean, precise, and authoritative.

When they are exposed, it becomes clear that:
* different assumptions produce different answers,
* confidence depends on the weakest element,
* and precision without context is meaningless.

This is not a flaw of the method.
It is the nature of geometry.


## Where tools hide the problem

At this point, a reasonable question arises:

*If pixel-to-ground mapping relies on so many assumptions, why does it usually look fine?*

The short answer is: because tools have to decide something.

Modern geospatial tools are not careless. They are designed to be usable, responsive, and productive. Faced with missing or ambiguous information, they do what any practical system must do: they choose sensible defaults.

That is not a flaw.
It is a necessity.

### Defaults are answers to unasked questions

Every time a tool places a point on the ground from imagery, it implicitly answers questions such as:
* which surface should be used?
* which altitude reference applies?
* which projection should express the result?
* which intersection should be considered “the” location?

Most of the time, these answers are reasonable.
They are chosen to work well in common situations.

The issue is not that these defaults are wrong.

The issue is that the questions themselves remain invisible.

### Precision hides uncertainty

Once a choice is made, tools present the result as a clean, precise coordinate:
* a point snaps to a map
* numbers are displayed with many decimals
* layers align visually

Nothing in the interface suggests:
* which assumptions were made
* how sensitive the result is to them
* or how much uncertainty remains

The result looks authoritative not because it is exact, but because uncertainty has been flattened away.

This is not deception.
It is the cost of abstraction.


### Abstractions work — until context changes

Defaults work remarkably well as long as the context matches the assumptions they were designed for:
* relatively flat terrain
* moderate accuracy requirements
* visual or qualitative use cases

Problems appear when the context shifts:
* terrain relief becomes significant
* vertical accuracy matters more than horizontal alignment
* relative position is more important than absolute location
* errors accumulate across processing steps

In those situations, the same defaults do not fail loudly.
They fail **quietly**.


## Error budgets: when reasonable assumptions accumulate

Up to this point, none of the assumptions we have discussed is outrageous.

Individually, they are all reasonable:
* collapsing a pixel into a single viewing direction,
* assuming a smooth or flat surface,
* trusting a camera pose estimate,
* choosing a default altitude reference,
* expressing the result in a convenient projection.

Most systems rely on these assumptions because they work well most of the time.

The problem appears when these assumptions are not treated as approximations, but as facts — and when their effects are allowed to accumulate.

### Errors rarely come from one place

Geospatial systems rarely fail because of a single, large mistake.
They fail because many small approximations quietly stack up.

Each step in a pixel-to-ground conversion contributes a bit of uncertainty:
* the pixel footprint introduces scale ambiguity,
* the camera pose introduces angular uncertainty,
* altitude introduces vertical uncertainty,
* terrain models introduce spatial uncertainty,
* reference frames introduce distortion and convention.

None of these errors is catastrophic on its own.
Together, they consume what engineers call an error budget.

When that budget is exhausted, results can still look precise — but no longer be reliable

### Why accumulated errors are hard to see

Stacked errors have an uncomfortable property: they do not announce themselves.

They often produce results that are:
* smooth,
* consistent,
* visually aligned,
* numerically precise.

Nothing obviously breaks.

Instead, the error shows up:
* when results are compared across datasets,
* when relative positions matter,
* when the same object is observed from different viewpoints,
* or when outputs are reused outside their original context.

By then, the assumptions that caused the error are long forgotten.

### Error budgets are about intent, not perfection

Thinking in terms of error budgets does not mean eliminating uncertainty.

It means being explicit about:
* where precision is required,
* where approximation is acceptable,
* and where assumptions are being spent.

A system that acknowledges its error budget can degrade gracefully.
A system that ignores it can fail silently.


## When the simplification is acceptable

Not every application needs geometric rigor.

In many cases, treating a pixel as a point is a reasonable and conscious tradeoff.

This is typically acceptable when:
* the goal is visual exploration or qualitative analysis,
* errors are small relative to the scale of interest,
* exact positioning has no legal or safety consequences,
* results are not reused as ground truth.

Examples include:
* city discovery or mapping apps,
* rough tagging or annotation,
* exploratory data analysis,
* early-stage prototyping.

In these contexts, simplicity is not a flaw — it is a feature.

The key condition is that the limitation is understood: *“This is approximate, and that’s fine.”*


### And when it is not

The same simplification becomes problematic when context changes.

Treating pixels as points is no longer acceptable when:
* relative position matters more than visual alignment,
* vertical accuracy is critical,
* results feed into downstream decisions,
* errors have legal, financial, or safety implications.

Examples include:
* infrastructure or asset mapping,
* land surveying and boundary definition,
* change detection over time,
* any workflow where outputs are reused as reference data

In these situations, the cost of a silent assumption can be much higher than the cost of additional complexity : A 1% error on the location of your mailbox relative to your front yard will maybe lead the mailman 10cm away from your mailbox, the same 1% error on a 1 km wide airfield could lead an aircraft with an automated landing system 10 meters outside the runway. Not the same implications and consequences.

What matters is not whether the error is large or small, but whether it is controlled.


## A final distinction to keep in mind

The difference between an acceptable approximation and a dangerous one is not magnitude.

It is awareness.

An explicit approximation can be managed, compensated for, or revisited. An implicit one cannot.

This is why understanding assumptions — and how they consume the error budget — is not an academic concern. It is a practical one.

A pixel is not a point.
It is a question: “Along this ray, where do you think the world is?”
Pixels have coordinates. Maps have coordinates.
But between the two lies geometry — and choices.
