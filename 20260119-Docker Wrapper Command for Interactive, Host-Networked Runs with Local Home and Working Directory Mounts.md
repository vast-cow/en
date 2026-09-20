---
title: "Docker Wrapper Command for Interactive, Host-Networked Runs with Local Home and Working Directory Mounts"
description: "Interactive and ephemeral execution: Runs an interactive container (-it) and automatically removes..."
pubDatetime: 2026-01-19T11:26:51.815Z
---

1. **Interactive and ephemeral execution:** Runs an interactive container (`-it`) and automatically removes it on exit (`--rm`).
2. **Host-like connectivity and filesystem context:** Uses the host network stack (`--network host`) and binds the user’s home directory into both the same path and `/root`, while setting `HOME` accordingly (`-v ... -e HOME=...`).
3. **Runs from the current directory and forwards inputs:** Sets the container working directory to the current directory (`-w "$PWD"`) and passes through any additional Docker args and image/command parameters (`"$@"`).

```bash
docker run -it --rm --network host -v "$HOME":"$HOME" -v "$HOME":/root -e HOME="$HOME" -w "$PWD" "$@"
```
