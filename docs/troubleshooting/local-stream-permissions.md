# Troubleshooting local stream permission errors

When an RStudio session starts or resumes, the launcher initializes a Unix
socket for intra-process communication. The socket path is computed in
`SessionPosixHttpConnectionListener` from the session context string and the
`RS_SESSION_TMP_DIR` environment variable:

- `local_streams::streamPath()` joins `RS_SESSION_TMP_DIR` with the stream file
  name and ensures the parent directories exist, logging any directory creation
  errors. 【F:src/cpp/session/include/session/SessionLocalStreams.hpp†L19-L38】
- The listener then binds to that path and logs the UID it will accept
  connections from, driven by `limitRpcClientUid`. 【F:src/cpp/session/http/SessionPosixHttpConnectionListener.cpp†L166-L193】

If `rsession` cannot create the stream directory or PID file you will see
errors like `Permission denied` or `No such file or directory` for paths under
`/var/run/rstudio-server/rstudio-rsession/<user>/...`. The most common causes
are:

1. `RS_SESSION_TMP_DIR` points to a directory owned by a different UID/GID than
   the one that is allowed to connect (the log line shows the enforced UID). The
   launcher writes files as the `rsession-run` UID, so stale directories owned by
   another user can block restarts.
2. The parent directory was cleaned up by the OS (e.g., `/var/run` on restart)
   and the process lacks permission to recreate it.

To mitigate:

- Confirm `RS_SESSION_TMP_DIR` resolves to a writable path for the launcher UID
  (often `rstudio-server` or the UID shown in the log).
- Remove stale per-user stream directories if they are owned by the wrong UID,
  then let the launcher recreate them.
- For Kubernetes pods that recreate `/var/run`, ensure an init container or
  startup hook creates the base directory with the correct owner and mode
  (`0777` as set by `initializeStreamDir`).
