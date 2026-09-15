# Changelog

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1

- The interface: a decoded image as dimensions, a colour model and a
  flat buffer; `decode` dispatching on the bytes' magic and `encode` on
  a format named with its own options; PNG and QOI through png-nv and
  qoi-nv, JPEG declared here; and resize, fit, crop, rotate, flip,
  overlay and convert.
- Every body is `todo()`. Nothing is implemented.
