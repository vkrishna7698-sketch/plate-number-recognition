# License Plate Recognition — OpenCV + Tesseract OCR

A pipeline that takes a photo of a car and reads the license plate number
out of it automatically.

**Result: correctly read the plate `HR26DK8337` from a real car photo.**

---

## What is this doing?

Given a photo of a car, the goal is to end up with plain text — the actual
plate number — without a human typing it in by hand. This is a common
building block for things like parking systems, toll booths, and traffic
enforcement cameras.

The hard part isn't reading text once you have a clean, isolated image of
just the plate — Optical Character Recognition (OCR) is a solved problem
for that case. The hard part is everything *before* OCR: finding exactly
where the plate is in a full photo of a car, cropping it out cleanly, and
cleaning it up enough that the OCR engine can actually read it correctly.

## The pipeline, step by step

1. **Resize** — the image is resized to a consistent width (600px) so every
   step downstream behaves predictably regardless of the original photo's
   resolution.

2. **Grayscale** — color isn't needed to find edges or text, so converting
   to grayscale simplifies every step that follows and speeds things up.

3. **Noise reduction (bilateral filter)** — smooths out small noise and
   texture (like the grille pattern or paint texture) while keeping actual
   edges sharp. Regular blurring would soften the plate's edges too, which
   would hurt detection.

4. **Edge detection (Canny)** — highlights boundaries in the image, turning
   the photo into a sketch of just its outlines. The plate's rectangular
   border is one of these edges.

5. **Contour finding** — an outline (edge) is one thing, but a *contour* is
   a full closed shape traced by following those edges. This step converts
   scattered edges into wholeconnected shapes.

## Two approaches to actually locate the plate

### Approach 1: Geometric shape detection

The idea: a license plate is a rectangle, so look through all the contours
found in the image, and find one that:
- Has exactly 4 corners (`cv2.approxPolyDP` simplifies a contour down to its
  corner points, and the code checks for exactly 4)
- Has a width-to-height ratio between 2.0 and 6.0 (a real plate is much
  wider than it is tall, unlike a window or headlight)

This is a very sensible first approach and works well in a lot of tutorials,
but real photos are messier — grilles, headlights, and background clutter
can all produce 4-sided shapes with a plausible aspect ratio, and lighting
or angle can distort the plate's actual outline enough that it doesn't come
out as a clean rectangle.

### Approach 2: White mask detection (the one that actually worked)

License plates (at least in this dataset/photo) are a solid bright white
background. Instead of relying on shape alone, this approach directly looks
for a bright, rectangular *region*:

1. **Threshold** — turn every pixel brighter than 170 white, everything else
   black. This isolates bright regions (like the plate) from the rest of
   the image.
2. **Morphological close, then open** — "close" fills in small gaps/holes
   inside the white region (letters make dark holes in an otherwise white
   plate); "open" removes small stray white specks elsewhere in the image
   that aren't the plate.
3. **Find contours in that clean mask**, and pick the one that's wide,
   large enough in area, and has a realistic plate-like aspect ratio.

This approach turned out to be more reliable than pure geometric shape
detection because it uses a property specific to plates (their brightness)
rather than relying purely on their outline, which is more fragile to
detect correctly in a cluttered photo.

## Cleaning up the crop before OCR

Once the plate region is cropped out:
- **Upscale 4x** (`cv2.resize` with cubic interpolation) — Tesseract reads
  small, low-resolution text poorly; scaling up gives it more pixels to
  work with per character
- **Otsu's thresholding** — automatically picks the best black/white cutoff
  point for this specific crop, turning it into crisp black text on a white
  background, which is the ideal input for OCR

## Reading the text (OCR)

Tesseract is run with a few specific settings:
- `--psm 7` — tells Tesseract to expect a single line of text (not a whole
  paragraph), which matches how a plate is laid out
- `-c tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789` —
  restricts recognition to only letters and digits, since a plate never
  contains punctuation or symbols; this alone removes a lot of common OCR
  misreads

## Results

**Input:** a photo of a car with an Indian license plate
**Output:** `HR26DK8337` — correctly read, matching the actual plate

![Original image](original_image.png)
*(TBD — add the saved original car photo)*

![Detected plate region](detected_plate.png)
*(TBD — add the image showing the plate outline detected)*

![Final cropped plate](cropped_plate.png)
*(TBD — add the final cleaned-up crop that was fed into OCR)*

## Tech stack

Python · OpenCV · imutils · Tesseract OCR (pytesseract) · NumPy

## Limitations

- Relies on the plate being a bright white background — plates with dark
  or colored backgrounds (common outside India) would need a different
  detection strategy
- Threshold value (170) and aspect ratio ranges are tuned to this specific
  photo's lighting and plate proportions; a different angle, lighting
  condition, or plate size could require re-tuning
- No handling for tilted/rotated plates — a plate photographed at an angle
  would need a perspective transform (`cv2.warpPerspective`) to straighten
  before OCR
- Tested on a single image; hasn't been validated across a larger dataset
  of varied cars, plate styles, or lighting conditions
- The geometric (4-sided contour) approach is left in the code as a first
  attempt, even though the white mask approach is what actually produces
  the final result
