---
name: decode-ticket-qr-from-photo
description: How to decode a guest-page QR from a photographed valet ticket on macOS — no zbarimg/pyzbar installed; use a throwaway Swift script with CoreImage CIDetector
metadata:
  type: reference
---

Redirect requests usually arrive as a **photo of a printed valet ticket** ("this UUID is linked to Operator 98 Location 400, I need tickets 51500-51999 to hit Location 410") without the source UUID written out. The UUID is only in the QR.

`zbarimg`, `pyzbar` and `cv2` are **not installed** on this machine and don't need to be. `swift` / `swiftc` are at `/usr/bin/`, so a throwaway script using CoreImage's `CIDetector(ofType: CIDetectorTypeQRCode)` decodes it with no dependencies. Vision's `VNDetectBarcodesRequest` works too as a cross-check.

Even a dark, blurry, skewed phone photo decoded on the **raw** image on the first try — don't preprocess before trying raw. Useful variants if raw fails: 2x/3x/4x upscale, `CIColorControls` contrast 1.5-4.0 with saturation 0, `CIUnsharpMask`.

Payload format is `URL:https://guest.corepark.com/ticket/<location-uuid>/<ticket-number>` — note the `URL:` prefix CIDetector prepends.

**Why:** Saves a round-trip asking the user to type the UUID, and the printed ticket number in the photo confirms the range they're describing.

**How to apply:** Write the script to the scratchpad and run `swift qr.swift <image-path>`. The decoded UUID is the `source_location_uuid` for the new rule in `uuidSetter()` — see [[uuid-redirect-flow-origin-backend-migration]]. The **target** UUID cannot be derived anywhere locally (the frontend never sees operator/location IDs, and the backend `location_redirect` resolver still doesn't exist) — always ask the user for it, or `SELECT location_uuid FROM company.parking_location WHERE operator_company_id = ? AND parking_location_id = ?`.
