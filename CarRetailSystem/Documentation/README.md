# CarRetailSystem — Modernization Documentation

This folder contains the full analysis and modernization plan for the CarRetailSystem legacy application.

## Documents

| File | Contents |
|------|----------|
| [01_CURRENT_STATE.md](01_CURRENT_STATE.md) | Technology stack, architecture, code inventory, business domain |
| [02_SECURITY_VULNERABILITIES.md](02_SECURITY_VULNERABILITIES.md) | All identified vulnerabilities with locations and severity |
| [03_TARGET_ARCHITECTURE.md](03_TARGET_ARCHITECTURE.md) | Recommended modern architecture and technology stack |
| [04_MIGRATION_STRATEGY.md](04_MIGRATION_STRATEGY.md) | Strangler Fig migration plan, phases, timelines |
| [05_QUICK_WINS.md](05_QUICK_WINS.md) | Immediate fixes applicable to the legacy system today |

## Summary

The CarRetailSystem is a 15+ year old monolithic car retail sales application built on Classic ASP, VB6 COM components, and SQL Server 7.0. It is in maintenance mode and carries multiple critical security vulnerabilities including pervasive SQL injection and hardcoded database credentials.

**Recommended path:** Rebuild as a modular monolith using ASP.NET Core 8 + React, migrated incrementally via the Strangler Fig pattern over ~14 weeks.

---

*Analysis date: April 2026*
