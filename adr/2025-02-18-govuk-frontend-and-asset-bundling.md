# GOV.UK Frontend and asset bundling

In our current systems, we download the GOV.UK Frontend zip file, unpack it, and then serve those pre-compiled assets directly. This makes it difficult to extend or modify those styles and components, but provides a simple developer experience.

We expect, as we develop a new integrated service, to need to override some variables that GOV.UK Frontend exposes or to build components and styles that extend what is provided natively. Doing this easily and effectively necessitates, to some degree, compiling and bundling GOV.UK Frontend from source.

There are a huge number of ways this could be achieved, as the JS/frontend web ecosystem tends to be fairly sprawling.

A few options are presented below to deal with our frontend JS/CSS needs.

## Options

### 1: Download GOV.UK Frontend zip, using the pre-compiled static files and assets

### 2: Use eg DartSass to compile only CSS; reuse GOV.UK Frontend JS files directly

### 3: Use a bundling system, eg Vite, to compile both CSS and JS from GOV.UK Frontend src.

## Decision

We will go for option 3, using Flask-Vite, for now. This should provide a clean and relatively simple developer experience, allowing us to extend and adjust GOV.UK Frontend styles and components as needed.

If it ends up adding undue complexity, or if we don't use it, we can fall back to either option 1 or 2.
