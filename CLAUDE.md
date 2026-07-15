# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

SICA (Sistema de Inventario y Control de Activos) is the front-end of an asset control/management web app. It's a **static, no-build vanilla HTML/CSS/JavaScript site** — there is no `package.json`, bundler, transpiler, package manager, linter, or test runner. Pages are opened directly in the browser (or served via a static server like VS Code's Live Server, configured in `.vscode/settings.json` on port 5503).

The front-end talks to a separate backend (not in this repo) over plain `fetch()` calls to `http://127.0.0.1:8000` / `http://localhost:8000` (both forms are used interchangeably across files), which persists data in MongoDB. There is no `.env`/config layer — API base URLs are hardcoded per-call in each `js/data_access/*.js` file.

## Running the app

There is no build or dev-server command to run. To work on the app:
- Open an `html/*.html` file directly in a browser, or
- Serve the repo root with any static file server (e.g. VS Code Live Server on port 5503, per `.vscode/settings.json`)
- Ensure the backend API is running locally on port 8000, since nearly every page fetches data on load

There are no automated tests, lint config, or CI in this repo. Verify changes by loading the relevant page in a browser and exercising the flow manually.

## Architecture

### Directory layout
- `html/` — all app pages (the actual product UI). Each page is a standalone HTML document with `<html id="pageName">`, used by scripts to detect which page is currently loaded (see below).
- `css/` — shared stylesheets, split by page family: `general_style.css` (header/nav/layout shared everywhere), `form_pages.css`, `table_pages.css`, `signin_signup_pages.css`.
- `js/data_access/` — one file per page, named to match the HTML page (e.g. `assets.html` ↔ `assets.js`). Contains: `fetch()` calls to the backend API, DOM element lookups (`document.getElementById`/`querySelector`), and event listeners wiring the two together. This is effectively the "controller" layer per page.
- `js/dom_manipulation/` — cross-page utilities shared across many pages: `form_validation.js` (generic form validation engine), `sweet_alert.js` / `user_feedback.js` (alert/toast helpers), `disable_enable_input_fields.js`, `upload_img_display.js`.
- `svg/`, `images/` — static assets.
- `landing_page_sica/` and `landing_page_team/` — separate, self-contained marketing/landing sites with their own `css/`, `js/`, `images/`/`assets/`, unrelated to the main app's data flow.

### Page ↔ script wiring convention
Every app page in `html/` loads its scripts at the end of `<body>`, always in roughly this order:
1. `js/data_access/session_logic.js` (or `index.js` on the home page) — reads `sessionUserData` from `localStorage`, applies role-based UI (hides/shows nav tabs), redirects to sign-in if there's no session.
2. `js/dom_manipulation/form_validation.js` — only on pages with a `<form>`.
3. `js/dom_manipulation/sweet_alert.js` and `user_feedback.js` — shared alert helpers, loaded on nearly every page.
4. `js/data_access/<page-name>.js` — the page-specific controller (fetches + builds the page).

Because none of this uses modules/bundling, **script load order matters** — later scripts freely reference `const`s and functions defined in earlier ones (e.g. `session_logic.js` defines `sessionUserData` that page scripts like `assets.js` read directly; `form_validation.js` defines `form`/`formInputs` used across pages). When adding a new page or script, follow the existing `<script>` ordering in a similar page rather than inventing a new one.

### Session & auth
- Login (`js/data_access/signin.js`) POSTs credentials to the backend, stores the returned user object as JSON in `localStorage['sessionUserData']`, then redirects to `html/index.html`.
- `session_logic.js` runs on every authenticated page: it parses `sessionUserData` from `localStorage`, and branches UI visibility (which nav tabs/report pages are shown) based on `sessionUserData.role`. Known roles include `Encargado de Inventario por Unidad`, `Proveeduría`, and `Jefatura`, each with different nav/report tab access.
- There is no token/session expiry handling beyond checking `localStorage` on each page load; logout simply clears `sessionUserData` and redirects to `signin.html`.
- Selected entity IDs (e.g. the asset a user picked from a table) are passed between pages via `localStorage` (e.g. `storeAssetId` in `assets.js`), not query params.

### Forms & validation
`js/dom_manipulation/form_validation.js` is a single generic engine shared by all forms:
- `validationFields` is a lookup keyed by `<form id>Fields`, listing each field's id and whether it currently passes validation.
- `validateForm` is attached to every input's `blur`/`keyup`/`click` events and dispatches to per-type validators (`validateEmptyField`, `validateLettersField`, `validateNumbersField`, `validateEmailField`, `validateFileField`) based on the input's element `id`.
- Page-specific submit handlers (in the matching `js/data_access/*.js` file) gate submission on `Object.values(validationFields['<formId>Fields']).every(Boolean)`.
- When adding a new form: give the `<form>` an `id`, add a matching `<id>Fields` object to `validationFields`, add a `case` to `validateForm`'s switch for each new field id, and add matching `<fieldId>Error` elements in the HTML for error messages (toggled via the `formInputErrorActive` class).

### Tables (list pages)
Pages like `assets.html`, `units.html`, `users.html` share a common pattern implemented per-page in their `data_access` file: fetch a list from the backend, render `<tr>`s into a `<table>`, support sort (radio buttons), filter (`<select>`s), search (text inputs), row selection (radio buttons per row), and client-side pagination (`pagination()` — fixed page size of 11 rows). When building a new list page, copy this pattern from the closest existing page (e.g. `assets.js`) rather than reimplementing pagination/sorting from scratch.

### Naming/language conventions
- UI text and user-facing error messages are in Spanish; code (variable/function names, comments) is in English.
- Page HTML ids (e.g. `id="assets"`, `id="myprofile"`) are used at runtime as page-detection flags in shared scripts (`pageId = document.querySelector('html').id`) — keep these unique and stable when renaming pages.
