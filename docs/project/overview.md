## Scope
This file owns only:
- A concise project overview for agents.
- Stable high-level project facts shared by backend and frontend.

This file must not contain:
- Detailed domain rules, implementation plans, or module-specific behavior.

Source of truth: reference

# Project Overview

YM Tax CRM is an internal staff CRM for clients, binders, billing, charges, tax workflows, VAT reports, annual reports, reminders, notifications, signing, correspondence, and operational work tracking.

The backend is FastAPI, SQLAlchemy ORM, Alembic, and Pydantic v2.

The frontend is React, TypeScript, Vite, Tailwind CSS, React Query, Zustand for auth/session state, react-hook-form, Zod, Axios, and React Router.

The UI is Hebrew-first and RTL-aware. User-facing UI copy should be Hebrew where relevant.

The system is still in development unless explicitly stated otherwise. Do not assume real production users or production data.
