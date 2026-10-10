# Third-party notices

This plugin's grant model (capability grammar, narrowing resolution, gating,
depth bounds, grant-via-env propagation) is a semantic port of:

- **pi-daddy** — Copyright (c) NousResearch — MIT License
  (`npm pack pi-daddy`; `dist/kernel/{capabilities,resolve,propagation,spawn}.js`)

No pi-daddy code is vendored; the model is reimplemented in Rust for the gray
plugin wire.
