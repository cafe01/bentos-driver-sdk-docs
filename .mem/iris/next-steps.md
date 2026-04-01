# Driver SDK Docs — Next Steps

## Ready to write when the time comes

- **Build an Inference Driver tutorial**: P4 deep-dive covers the pattern; tutorial needs a real provider integration to walk through.
- **Deploy pipeline**: GitHub Pages via `.github/workflows/deploy.yaml` — already in place.
- **Mobile responsiveness**: Sidebar currently hidden on mobile (CSS media query). Could add a hamburger menu.
- **Search**: Hugo-native search or Algolia when content grows.

## Content gaps identified

- No "Contributing a Driver" guide yet (how to package, distribute, register with bentosd)
- No cross-language comparison page (when Py/Rust/Go bindings exist)
- Wire protocol page could include actual protobuf schema excerpts once fuse_wire.proto is public
- No "Debugging Drivers" guide (playground_driver as diagnostic tool)

## Not my domain but noted

- The taxonomy warning in hugo build is benign (we don't use tags/categories). Could add empty taxonomy layouts to silence it.
