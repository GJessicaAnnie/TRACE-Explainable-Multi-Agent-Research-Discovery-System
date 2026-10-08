# TRACER Development Guidelines

## Objective

Build a production-ready MERN application following clean architecture, reusable components, and maintainable code.

---

## Tech Stack

Frontend
- React (Vite)
- JavaScript
- Tailwind CSS v4
- React Router

Backend
- Node.js
- Express

Database
- MongoDB Atlas

Future
- Socket.io
- OpenAlex API
- Semantic Scholar API

---

# Coding Principles

- Keep components small and reusable.
- One responsibility per component.
- Prefer composition over duplication.
- Avoid unnecessary abstraction.
- Use semantic HTML.
- Write readable code.

---

# Folder Structure

Follow the project folder structure exactly.

Do not create unnecessary folders.

---

# React Guidelines

Use Functional Components only.

Use Hooks.

No Class Components.

Prefer custom hooks when logic is reused.

---

# Styling

Use Tailwind CSS.

Avoid inline styles.

Avoid custom CSS unless necessary.

Use utility classes.

---

# Naming

PascalCase

Components

Navbar.jsx

Hero.jsx

SearchBar.jsx

camelCase

Variables

Functions

Constants

UPPER_CASE

---

# Responsiveness

Desktop First

Tablet

Mobile

Every component must be responsive.

No horizontal scrolling.

---

# Performance

Avoid unnecessary re-renders.

Keep components lightweight.

Lazy load only when necessary.

---

# Accessibility

Semantic HTML.

Proper headings.

Buttons should have labels.

Inputs should have placeholders and labels where needed.

---

# Reusability

If a UI pattern repeats more than once, convert it into a reusable component.

---

# UI Rules

Follow docs/ui_philosophy.md.

Do not create generic SaaS layouts.

Keep the interface clean.

Large whitespace.

Minimal animations.

Purposeful interactions.

---

# Code Quality

Readable.

Modular.

Maintainable.

Production ready.

Avoid hacks.

Avoid duplicate code.

---

# Don't

Do not use Bootstrap.

Do not use Material UI.

Do not use jQuery.

Do not use inline CSS.

Do not use hardcoded pixel values unless required.

Do not create unnecessary dependencies.

---

# Future Compatibility

Frontend should be designed so backend integration can happen without major refactoring.

All API calls will be added later.

Current phase focuses only on UI implementation.