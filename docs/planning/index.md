# Implementation Plan Index

Use this file to orchestrate active ticket plans with clear order and dependency.

## Status Legend

- `DRAFT`: decisions not locked, execution blocked
- `REVIEW`: decisions locked, waiting for user manual approval
- `READY`: user approved manually, execution allowed
- `DONE`: implemented and verified

## Execution Order

_Tidak ada ticket aktif. Ticket selesai dipindah ke `Ticket-Implemented/`._

## Global Agent Rules

1. Always start from the first ticket not marked `DONE`.
2. Never execute a ticket with status `DRAFT` or `REVIEW`.
3. Read each ticket's `Business Decision Snapshot` before any code change.
4. Follow each ticket's `Non-Negotiable Technical Contract` exactly.
5. Stop and report if a dependency ticket is incomplete or inconsistent.
