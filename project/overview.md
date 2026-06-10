## Scope
This file owns only:
- A concise project overview for agents.
- Stable high-level project facts shared by backend and frontend.

This file must not contain:
- Detailed domain rules, implementation plans, or module-specific behavior.

Source of truth: reference

# Project Overview

YM Tax CRM is an internal staff CRM for clients, binders, billing, charges, tax workflows, VAT, annual reports, reminders, notifications, signing, communications, and operational work tracking.

The staff-facing roles are `ADVISOR` for full access and `SECRETARY` for operational access with limited write permissions.

The backend is Python 3.12+, FastAPI, Uvicorn, SQLAlchemy 2.0 ORM, Alembic, and Pydantic v2.

Backend authentication uses JWT HS256, bcrypt password hashing, and `token_version` invalidation.

Backend storage is local filesystem in development and test, and Cloudflare R2 in staging and production.

Deployment is on Render through `render.yaml`.

The frontend is React, TypeScript, Vite, Tailwind CSS, React Query, Zustand for auth/session state, react-hook-form, Zod, Axios, and React Router.

The UI is Hebrew-first and RTL-aware. User-facing UI copy should be Hebrew where relevant.

The system is still in development unless explicitly stated otherwise. Do not assume real production users or production data.
