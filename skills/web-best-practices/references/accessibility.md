# Accessibility checklist

This reference lists accessibility items to verify during a review. It is based on WCAG principles and should be validated with official sources where compliance is implied.

## Images

- Informative images have a descriptive `alt` attribute.
- Decorative images use an empty `alt=""`.

## Contrast

- Text contrast meets WCAG AA ratios where applicable (commonly 4.5:1 for normal text, 3:1 for large text).
- Interactive elements and focus indicators are distinguishable.

## Structure

- Pages use a logical heading hierarchy (`h1`, `h2`, etc.).
- Landmarks and semantic HTML are used where appropriate.

## Forms

- Each interactive input has an associated label.
- Errors are identifiable visually and programmatically.

## Keyboard and focus

- Core functionality is reachable by keyboard.
- Focus is visible and predictable.

## Meaning

- Information is not conveyed by color alone.
- Instructions and status changes are clear.

## Media and timing

- Content does not rely solely on timing where user control matters.
- Auto-playing media, if present, is controllable.

## References

- WCAG 2.2: https://www.w3.org/TR/WCAG22/

## Notes

- Accessibility should be verified with a combination of automated tools and manual review.
- Exact target level should be set based on the project’s accessibility requirements.
