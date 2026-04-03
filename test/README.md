# `test/` — test fixtures and temporary packaging directories

This folder contains various test fixtures and temporary directories used by parts of vtiger tooling.

Subfolders in this repo:
- `contact/`, `product/`, `user/`, `logo/`, `upload/` — sample assets/fixtures used by demos/tests
- `wordtemplatedownload/` — sample/fixture area for word template download flows
- `vtlib/` — vtlib-related temporary directory (used for module packaging/export in some flows)

Notes:
- This is not a modern unit-test suite; it is primarily **fixture data** and **tooling temp space**.
- vtlib’s local note: see `test/vtlib/README.txt`.
