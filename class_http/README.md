# `class_http/` — legacy HTTP client helper

This directory provides a lightweight HTTP helper used by some integration features to fetch remote content.

Key files:
- `class_http.php` — HTTP utility class (request/response handling)
- `image_cache.php` — helper related to caching fetched images/content

## Notes for developers
- This is used in specific integrations and older vtiger features.
- For modern PHP projects you might use cURL directly, but in this codebase prefer existing utilities for consistency unless refactoring intentionally.
