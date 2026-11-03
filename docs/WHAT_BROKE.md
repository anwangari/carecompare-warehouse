# What Broke

One entry per session, written in my own words, at the end of the session.
Short is fine. Honest is required.

Format:

```
## Session N · YYYY-MM-DD
**Broke:**  what went wrong, as I first saw it
**Cause:**  what was actually going on
**Fix:**    what I changed
**Lesson:** what I would do differently next time
```

---

<!-- First entry goes here after session 1. -->

## Session 1 | Setting Up Airflow in Docker | 2026-11-02 
**Broke:**  ```docker compose up airflow-init``` failing .
**Cause:**  No idea: one minute the pull reached Docker Hub, the next minute it didn't
**Fix:**    I downloaded a fresh image separately with ```docker pull apache/airflow:3.3.2```, ran compose again
**Lesson:** Create a docker-compose.yml with healthchecks to ensure all services are running straight as expected. If a service is missing, ```docker compose -d``` fails loudly until all checks are green.


---
