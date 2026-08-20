# Improvement Roadmap Ranked by ROI

## Highest ROI

1. Detection-only public demo

   Cheap to host, shows the most reliable part of the system, and uses small AASIST checkpoints.

2. Persistent classification history

   Replace in-memory history with SQLite or Postgres so demos survive restarts.

3. Safer upload handling

   Add MIME sniffing, duration limits, stronger file validation, request throttling, and per-user cleanup.

4. Clear model status UX

   Keep generation disabled when XTTS is unavailable and show classifier readiness clearly.

5. Basic CI and smoke tests

   Prevents broken imports, missing docs, and missing checkpoint files from reaching GitHub.

## Medium ROI

6. Async generation queue

   Move XTTS generation to a background worker with Redis/RQ, Celery, Dramatiq, or a managed queue.

7. Object storage

   Store uploads and generated audio in S3/GCS/R2 instead of local disk.

8. Better result history

   Add downloadable reports, per-file score explanations, and user-facing confidence context.

9. Batch classification

   Let users upload multiple clips for a table of results.

10. Model comparison view

   Show Model A vs Model B agreement, disagreement, scores, and thresholds in a compact table.

## Lower ROI Until the Demo Is Stable

11. Authentication

   Useful for public production, but not necessary for a limited portfolio demo.

12. Experiment tracking

   Helpful for continued research; use MLflow, Weights and Biases, or simple JSON registry.

13. Explainability dashboard

   Existing scripts can generate temporal, sub-band, and surrogate outputs. A UI can come after stable inference.

14. Monitoring

   Add request metrics, latency metrics, GPU memory reporting, and error tracking after deployment.

15. Full GPU generation service

   Highest demo impact but also highest hosting cost because XTTS checkpoints are multi-GB.

