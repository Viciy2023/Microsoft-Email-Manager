FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    DEBIAN_FRONTEND=noninteractive \
    HOME=/home/user \
    PATH=/home/user/.local/bin:$PATH

RUN apt-get update \
    && apt-get install -y --no-install-recommends git ca-certificates bash coreutils \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -u 1000 user

WORKDIR /workspace

ARG GIT_REPO=https://github.com/Maishan-Inc/Microsoft-Email-Manager.git
RUN git clone --depth 1 "$GIT_REPO" Microsoft-Email-Manager \
    && chown -R user:user /workspace

USER user
WORKDIR /workspace/Microsoft-Email-Manager

RUN pip install --upgrade pip \
    && pip install -r requirements.txt \
    && pip install "huggingface_hub[cli]"

RUN python - <<'PYCODE'
from pathlib import Path

script = r'''#!/usr/bin/env bash
set -euo pipefail

APP_DIR="/workspace/Microsoft-Email-Manager"
LOCAL_DATA_DIR="${LOCAL_DATA_DIR:-/workspace/data}"
HF_BUCKET="${HF_BUCKET:-DanielleNguyen/Microsoft-Email-Manager-Storage}"
FAST_SYNC_INTERVAL_SECONDS="${FAST_SYNC_INTERVAL_SECONDS:-15}"
SYNC_INTERVAL_SECONDS="${SYNC_INTERVAL_SECONDS:-300}"
STATE_DIR="${SYNC_STATE_DIR:-/tmp/hf-sync-state}"
LOCK_DIR="${SYNC_LOCK_DIR:-/tmp/hf-sync-lock}"
LAST_SYNC_FILE="$STATE_DIR/last-sync.sha256"

mkdir -p "$LOCAL_DATA_DIR" "$STATE_DIR"

export DATA_DIR="$LOCAL_DATA_DIR"
export ACCOUNTS_FILE="${ACCOUNTS_FILE:-$LOCAL_DATA_DIR/accounts.json}"
export AUTH_FILE="${AUTH_FILE:-$LOCAL_DATA_DIR/auth.json}"
export SESSIONS_FILE="${SESSIONS_FILE:-$LOCAL_DATA_DIR/sessions.json}"
export API_KEYS_FILE="${API_KEYS_FILE:-$LOCAL_DATA_DIR/api_keys.json}"
export PUBLIC_SHARES_FILE="${PUBLIC_SHARES_FILE:-$LOCAL_DATA_DIR/public_shares.json}"
export OPEN_ACCESS_SESSIONS_FILE="${OPEN_ACCESS_SESSIONS_FILE:-$LOCAL_DATA_DIR/open_access_sessions.json}"
export ADMIN_LOGIN_ATTEMPTS_FILE="${ADMIN_LOGIN_ATTEMPTS_FILE:-$LOCAL_DATA_DIR/admin_login_attempts.json}"
export ACCOUNT_HEALTH_FILE="${ACCOUNT_HEALTH_FILE:-$LOCAL_DATA_DIR/account_health.json}"
export ACCOUNT_CLASSIFICATIONS_FILE="${ACCOUNT_CLASSIFICATIONS_FILE:-$LOCAL_DATA_DIR/account_classifications.json}"
export EMAIL_TAGS_FILE="${EMAIL_TAGS_FILE:-$LOCAL_DATA_DIR/email_tags.json}"
export SITE_SETTINGS_FILE="${SITE_SETTINGS_FILE:-$LOCAL_DATA_DIR/site_settings.json}"
export TRUST_PROXY_HEADERS="${TRUST_PROXY_HEADERS:-true}"

log() {
  printf '[hf-sync] %s\n' "$*"
}

bucket_ready() {
  [ -n "${HF_TOKEN:-}" ] && [ -n "$HF_BUCKET" ]
}

compute_state() {
  if [ ! -d "$LOCAL_DATA_DIR" ]; then
    printf 'empty\n'
    return
  fi

  python - "$LOCAL_DATA_DIR" <<'PYHASH'
import hashlib
import sys
from pathlib import Path

base = Path(sys.argv[1])
if not base.exists():
    print('empty')
    raise SystemExit

entries = []
for path in sorted(base.rglob('*')):
    if path.is_file():
        rel = path.relative_to(base).as_posix()
        stat = path.stat()
        entries.append(f"{rel}|{stat.st_size}|{int(stat.st_mtime_ns)}")
print(hashlib.sha256('\n'.join(entries).encode('utf-8')).hexdigest())
PYHASH
}

read_last_state() {
  if [ -f "$LAST_SYNC_FILE" ]; then
    cat "$LAST_SYNC_FILE"
  fi
}

write_last_state() {
  printf '%s' "$1" > "$LAST_SYNC_FILE"
}

with_lock() {
  if mkdir "$LOCK_DIR" 2>/dev/null; then
    "$@"
    status=$?
    rmdir "$LOCK_DIR" >/dev/null 2>&1 || true
    return "$status"
  fi
  return 1
}

pull_bucket_to_local() {
  if ! bucket_ready; then
    return 0
  fi
  log "pulling bucket to local: $HF_BUCKET -> $LOCAL_DATA_DIR"
  hf buckets sync "hf://buckets/$HF_BUCKET" "$LOCAL_DATA_DIR"
}

push_local_to_bucket() {
  if ! bucket_ready; then
    return 0
  fi
  log "pushing local to bucket: $LOCAL_DATA_DIR -> $HF_BUCKET"
  hf buckets sync "$LOCAL_DATA_DIR" "hf://buckets/$HF_BUCKET"
}

initial_sync() {
  if ! bucket_ready; then
    log "HF_TOKEN or HF_BUCKET missing, skip startup bucket sync"
    write_last_state "$(compute_state)"
    return 0
  fi

  with_lock pull_bucket_to_local || true
  write_last_state "$(compute_state)"
}

fast_sync_loop() {
  while true; do
    sleep "$FAST_SYNC_INTERVAL_SECONDS"
    current_state="$(compute_state)"
    previous_state="$(read_last_state || true)"
    if [ "$current_state" != "$previous_state" ]; then
      if with_lock push_local_to_bucket; then
        write_last_state "$current_state"
      fi
    fi
  done
}

reconcile_once() {
  pull_bucket_to_local
  push_local_to_bucket
}

reconcile_loop() {
  while true; do
    sleep "$SYNC_INTERVAL_SECONDS"
    if with_lock reconcile_once; then
      write_last_state "$(compute_state)"
    fi
  done
}

shutdown() {
  log "shutting down"
  if bucket_ready; then
    with_lock push_local_to_bucket || true
  fi
  if [ -n "${APP_PID:-}" ]; then
    kill "$APP_PID" >/dev/null 2>&1 || true
  fi
  if [ -n "${FAST_SYNC_PID:-}" ]; then
    kill "$FAST_SYNC_PID" >/dev/null 2>&1 || true
  fi
  if [ -n "${RECONCILE_PID:-}" ]; then
    kill "$RECONCILE_PID" >/dev/null 2>&1 || true
  fi
}
trap shutdown EXIT INT TERM

initial_sync
fast_sync_loop &
FAST_SYNC_PID=$!
reconcile_loop &
RECONCILE_PID=$!

cd "$APP_DIR"
python main.py &
APP_PID=$!
wait "$APP_PID"
'''

path = Path('/home/user/.local/bin/start-hf-space.sh')
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(script, encoding='utf-8')
path.chmod(0o755)
PYCODE

EXPOSE 8000

CMD ["/home/user/.local/bin/start-hf-space.sh"]
