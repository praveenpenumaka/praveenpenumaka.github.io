# Can the Browser Render Gaussian Splats with Plain DOM Elements?

I've been reading a lot of the papers, experiments, and implementations coming out around Gaussian splatting. What keeps me interested is how quickly the field moves: a new technique appears, and within months people have made it faster, smaller, and better looking.

One recurring rendering bottleneck is sorting.

Gaussian splats are translucent, so they usually need to be rendered back-to-front for alpha compositing to produce the correct result. Sorting thousands, or even millions, of Gaussians whenever the camera moves can become expensive.

So I started wondering whether I could skip the explicit sort and let the browser handle compositing.

## The experiment

Browsers already layer and alpha-composite elements. A DOM element can have opacity, a background color, a transform, and a `z-index`.

That covers most of what a projected Gaussian needs:

- Position
- Width and height
- Rotation
- Color
- Opacity
- Depth

So I built a small experimental Gaussian splat renderer using plain browser elements.

You can try it here: <https://penumpra.com/dom-gaussian-splats/>

The experiment supports two renderers:

1. A renderer where every Gaussian is a `<div>`
2. A renderer where every Gaussian is an SVG `<ellipse>`

I wasn't trying to build the fastest Gaussian renderer; WebGL and WebGPU are the right tools for that. I just wanted to see how far the browser's existing layout and compositing could be pushed.

## Rendering with DIVs

For the DIV renderer, every Gaussian becomes an absolutely positioned element.

CSS handles its projected ellipse:

```css
transform: translate(-50%, -50%) rotate(...);
border-radius: 50%;
opacity: ...;
z-index: ...;
```

The Gaussian's scale and covariance set the ellipse's size and rotation. Spherical harmonics give it a view-dependent color, and the Gaussian's opacity becomes the element's opacity.

Instead of sorting every frame, I quantize the camera distance and convert it into a `z-index`. I never sort the splats in JavaScript; I just assign their depth and let the browser handle stacking and compositing.

Ordering isn't free, of course. The browser still has to resolve stacking and composite the elements. I just moved that work out of my renderer.

On my machine, the DIV renderer stayed usable up to around 40,000 splats. Past that, memory use and DOM update costs started to show.

That result was better than I expected.

## Rendering with SVG

SVG seemed like a natural fit at first. An SVG `<ellipse>` already represents the projected shape of a Gaussian, and SVG gives straightforward control over position, scale, rotation, color, and opacity.

SVG has one catch, though: normal SVG elements don't support `z-index` the way positioned HTML elements do.

They're painted in document order, so changing compositing order means reordering nodes or grouping them into ordered depth buckets.

I settled on 256 depth buckets. That avoids a full per-Gaussian sort, but the buckets still have to sit in the right order in the SVG tree.

In my tests, updating the SVG tree cost more than reusing the DIV pool.

## Reusing what I could

Early versions redid too much work every frame. I moved as much as I could into caches:

- The downloaded PLY buffer is parsed only once.
- The selected Gaussian sample is cached.
- Sampling happens again only when the requested count changes.
- Centered positions are cached.
- World covariance matrices are precomputed.
- Projected-point storage is reused.
- Render-packet objects are reused.
- DIV elements are allocated once and updated instead of recreated.
- Only the selected renderer remains connected to the live document.

Those changes helped, and they made the real bottleneck obvious: updating and compositing thousands of elements is expensive.

With the camera in auto-orbit, the best I saw was about 4 FPS. That will vary with the browser, machine, renderer, and splat count, but it gives a rough idea of the limits.

## What about the GPU?

WebGPU can speed up the calculations:

- Spherical-harmonic evaluation
- Camera transforms
- Covariance projection
- Ellipse calculation
- Visibility checks

WebGPU shaders can't touch DOM or SVG elements directly, though.

The results have to be copied back into JavaScript, which still has to update every element. The readback adds synchronization overhead, and the DOM bottleneck is still there.

Moving just the calculations to the GPU only helped a little. A real GPU renderer would draw directly into a canvas and skip the DOM, but that's a different experiment.

## Was it practical?

Not as a production renderer for millions of Gaussians. Still, it was a fun experiment, and I learned a lot from it.

It also showed me how layout, stacking, SVG ordering, JavaScript, GPU readback, and alpha compositing interact, and where the time goes.

It points to something I think could be useful, too: a lightweight preview renderer.

I'm thinking about packaging parts of it as a Rust library that generates SVG previews (or maybe animated GIFs) of large Gaussian models on a server. It wouldn't replace an interactive GPU renderer, but it could help with docs, model galleries, social previews, or anywhere a full renderer is overkill.

The source and live experiment are available here:

- **Demo:** <https://penumpra.com/dom-gaussian-splats/>
- **Source:** <https://github.com/praveenpenumaka/dom-gaussian-splats>

If you've tried something similar, or you work on Gaussian splatting or browser rendering, I'd be glad to hear about it.

**Note:** I took help from AI to edit this post.
