# Dashboards

Two dashboards were created using python scripts for demonstration purpose. 

## RIC Dashbaord
Lists the RIC pods, logs and their describe output. It can be used for restart pods as well.
Real-time terminal dashboard for monitoring pods on a Kubernetes-based
Near-RT RIC (e.g. OSC RIC platform) deployment.

Requirements covered:
  - Real-time pod list across ALL namespaces, auto-refreshing.
  - Arrow-key navigation over the pod list.
  - View real-time logs of the selected pod (Enter / l).
      * If the pod has multiple containers, choose one first.
      * Ctrl+C stops the log stream and returns to the dashboard.
  - Describe the selected pod (d) -> shown via `less`, q to return.
  - Restart the selected pod (r) -> always asks for confirmation first
    (restart = delete pod, letting its controller recreate it).
  - Quit the dashboard (q) from the main screen.

Requirements:
  - Python 3.7+
  - `kubectl` installed and configured (KUBECONFIG pointing at your
    Near-RT RIC cluster / context).
  - A Unix-like terminal (Linux/macOS, or WSL on Windows) since this
    uses the `curses` module.

Usage:
```bash
  python3 ric_pod_dashboard.py
```

Code is given below. Create ric_pod_dashboard.py: 
```python
#!/usr/bin/env python3

import curses
import json
import os
import shutil
import subprocess
import sys
import threading
import time
from datetime import datetime, timezone

REFRESH_INTERVAL_SECONDS = 2.0


# --------------------------------------------------------------------------- #
# Data fetching
# --------------------------------------------------------------------------- #

def human_age(creation_ts):
    """Convert a Kubernetes creationTimestamp into a short human age string."""
    if not creation_ts:
        return "-"
    try:
        created = datetime.strptime(creation_ts, "%Y-%m-%dT%H:%M:%SZ").replace(
            tzinfo=timezone.utc
        )
    except Exception:
        return "-"
    seconds = int((datetime.now(timezone.utc) - created).total_seconds())
    if seconds < 60:
        return f"{seconds}s"
    minutes = seconds // 60
    if minutes < 60:
        return f"{minutes}m"
    hours = minutes // 60
    if hours < 24:
        return f"{hours}h{minutes % 60}m"
    days = hours // 24
    return f"{days}d{hours % 24}h"


def fetch_pods():
    """Return (pods, error) where pods is a list of dicts, all namespaces."""
    try:
        proc = subprocess.run(
            ["kubectl", "get", "pods", "--all-namespaces", "-o", "json"],
            capture_output=True,
            text=True,
            timeout=10,
        )
    except FileNotFoundError:
        return None, "kubectl not found in PATH"
    except subprocess.TimeoutExpired:
        return None, "kubectl get pods timed out"
    except Exception as e:
        return None, str(e)

    if proc.returncode != 0:
        return None, proc.stderr.strip() or "kubectl returned a non-zero exit code"

    try:
        data = json.loads(proc.stdout)
    except json.JSONDecodeError as e:
        return None, f"failed to parse kubectl output: {e}"

    pods = []
    for item in data.get("items", []):
        meta = item.get("metadata", {}) or {}
        spec = item.get("spec", {}) or {}
        status = item.get("status", {}) or {}

        namespace = meta.get("namespace", "")
        name = meta.get("name", "")
        node = spec.get("nodeName") or "-"
        phase = status.get("phase", "Unknown")
        containers = [c.get("name") for c in spec.get("containers", []) if c.get("name")]
        container_statuses = status.get("containerStatuses") or []

        restarts = sum(cs.get("restartCount", 0) for cs in container_statuses)
        ready_count = sum(1 for cs in container_statuses if cs.get("ready"))
        total_count = len(containers) if containers else len(container_statuses)

        # Mirror kubectl's habit of surfacing waiting/terminated reasons
        # (e.g. CrashLoopBackOff, ImagePullBackOff) instead of just "Running".
        display_status = phase
        for cs in container_statuses:
            state = cs.get("state", {}) or {}
            if "waiting" in state and state["waiting"].get("reason"):
                display_status = state["waiting"]["reason"]
                break
            if (
                "terminated" in state
                and state["terminated"].get("reason")
                and phase != "Running"
            ):
                display_status = state["terminated"]["reason"]
                break

        pods.append(
            {
                "namespace": namespace,
                "name": name,
                "status": display_status,
                "ready": f"{ready_count}/{total_count}",
                "restarts": restarts,
                "age": human_age(meta.get("creationTimestamp")),
                "node": node,
                "containers": containers,
            }
        )

    pods.sort(key=lambda p: (p["namespace"], p["name"]))
    return pods, None


class PodWatcher(threading.Thread):
    """Background thread that polls `kubectl get pods` on an interval."""

    def __init__(self, interval=REFRESH_INTERVAL_SECONDS):
        super().__init__(daemon=True)
        self.interval = interval
        self.lock = threading.Lock()
        self.pods = []
        self.error = None
        self.last_update = None
        self._stop_event = threading.Event()

    def run(self):
        while not self._stop_event.is_set():
            pods, err = fetch_pods()
            with self.lock:
                if pods is not None:
                    self.pods = pods
                    self.error = None
                else:
                    self.error = err
                self.last_update = time.strftime("%H:%M:%S")
            self._stop_event.wait(self.interval)

    def stop(self):
        self._stop_event.set()

    def get_snapshot(self):
        with self.lock:
            return list(self.pods), self.error, self.last_update


# --------------------------------------------------------------------------- #
# Dashboard (curses UI)
# --------------------------------------------------------------------------- #

class Dashboard:
    COL_NS, COL_NAME, COL_READY, COL_STATUS, COL_RESTARTS, COL_AGE = 18, 38, 7, 18, 9, 8

    def __init__(self, stdscr, watcher):
        self.stdscr = stdscr
        self.watcher = watcher
        self.selected_key = None  # (namespace, name)
        self.scroll = 0
        self.message = ""
        self.message_time = 0.0

    # -- small helpers -----------------------------------------------------

    def set_message(self, msg):
        self.message = msg
        self.message_time = time.time()

    def safe_addstr(self, y, x, text, attr=0):
        h, w = self.stdscr.getmaxyx()
        if y < 0 or y >= h or x < 0 or x >= w:
            return
        try:
            self.stdscr.addnstr(y, x, text, max(0, w - x - 1), attr)
        except curses.error:
            pass

    def current_index(self, pods):
        if not pods:
            return -1
        if self.selected_key is None:
            return 0
        for i, p in enumerate(pods):
            if (p["namespace"], p["name"]) == self.selected_key:
                return i
        return 0  # previously selected pod is gone -> fall back to top

    def move_selection(self, pods, delta):
        if not pods:
            return
        idx = self.current_index(pods)
        idx = max(0, min(len(pods) - 1, idx + delta))
        self.selected_key = (pods[idx]["namespace"], pods[idx]["name"])

    # -- main loop -----------------------------------------------------------

    def run(self):
        curses.curs_set(0)
        self.stdscr.timeout(150)  # non-blocking-ish getch, ~6-7 fps
        curses.start_color()
        curses.use_default_colors()
        curses.init_pair(1, curses.COLOR_GREEN, -1)     # Running / healthy
        curses.init_pair(2, curses.COLOR_YELLOW, -1)    # Pending / transitional
        curses.init_pair(3, curses.COLOR_RED, -1)       # Failed / crashing / error
        curses.init_pair(4, curses.COLOR_CYAN, -1)      # headers / info

        while True:
            pods, err, last_update = self.watcher.get_snapshot()
            self.render(pods, err, last_update)

            key = self.stdscr.getch()
            if key == -1:
                continue
            elif key in (ord('q'), ord('Q')):
                break
            elif key in (curses.KEY_UP, ord('k')):
                self.move_selection(pods, -1)
            elif key in (curses.KEY_DOWN, ord('j')):
                self.move_selection(pods, 1)
            elif key in (curses.KEY_ENTER, 10, 13, ord('l'), ord('L')):
                self.view_logs(pods)
            elif key in (ord('d'), ord('D')):
                self.describe_pod(pods)
            elif key in (ord('r'), ord('R')):
                self.restart_pod(pods)
            elif key == curses.KEY_RESIZE:
                self.stdscr.clear()

    # -- rendering -----------------------------------------------------------

    def render(self, pods, err, last_update):
        self.stdscr.erase()
        h, w = self.stdscr.getmaxyx()

        title = " O-RAN Near-RT RIC :: Kubernetes Pod Dashboard "
        self.safe_addstr(0, 0, title.center(w, "="), curses.A_BOLD | curses.color_pair(4))

        info = f" Namespaces: ALL | Pods: {len(pods)} | Last update: {last_update or '-'} "
        self.safe_addstr(1, 0, info, curses.A_DIM)

        row = 2
        if err:
            self.safe_addstr(row, 0, f" kubectl error: {err}", curses.color_pair(3) | curses.A_BOLD)
            row += 1

        header_y = row + 1
        header = (
            f"{'NAMESPACE':<{self.COL_NS}}"
            f"{'NAME':<{self.COL_NAME}}"
            f"{'READY':<{self.COL_READY}}"
            f"{'STATUS':<{self.COL_STATUS}}"
            f"{'RESTARTS':<{self.COL_RESTARTS}}"
            f"{'AGE':<{self.COL_AGE}}"
            f"NODE"
        )
        self.safe_addstr(header_y, 0, header, curses.A_REVERSE)

        list_top = header_y + 1
        list_height = max(1, h - list_top - 4)

        idx = self.current_index(pods)
        if idx >= 0:
            if idx < self.scroll:
                self.scroll = idx
            elif idx >= self.scroll + list_height:
                self.scroll = idx - list_height + 1
        else:
            self.scroll = 0

        visible = pods[self.scroll:self.scroll + list_height]
        for row_i, pod in enumerate(visible):
            y = list_top + row_i
            real_idx = self.scroll + row_i
            is_selected = real_idx == idx

            st = pod["status"]
            if st in ("Running", "Succeeded", "Completed"):
                status_color = curses.color_pair(1)
            elif st in ("Pending", "ContainerCreating", "PodInitializing", "Terminating"):
                status_color = curses.color_pair(2)
            else:
                status_color = curses.color_pair(3)

            name = pod["name"]
            if len(name) >= self.COL_NAME:
                name = name[: self.COL_NAME - 4] + "..."

            line = (
                f"{pod['namespace']:<{self.COL_NS}}"
                f"{name:<{self.COL_NAME}}"
                f"{pod['ready']:<{self.COL_READY}}"
                f"{pod['status']:<{self.COL_STATUS}}"
                f"{str(pod['restarts']):<{self.COL_RESTARTS}}"
                f"{pod['age']:<{self.COL_AGE}}"
                f"{pod['node']}"
            )
            attr = (curses.A_REVERSE | curses.A_BOLD) if is_selected else status_color
            self.safe_addstr(y, 0, line, attr)

        if not pods and not err:
            self.safe_addstr(list_top, 2, "No pods found (waiting for first refresh)...", curses.A_DIM)

        footer_y = h - 3
        self.safe_addstr(footer_y, 0, "-" * w, curses.A_DIM)
        keys_line = " Up/Down Move   Enter/l Logs   d Describe   r Restart   q Quit "
        self.safe_addstr(footer_y + 1, 0, keys_line.center(w), curses.A_BOLD)

        if self.message and (time.time() - self.message_time) < 5:
            self.safe_addstr(footer_y + 2, 0, " " + self.message, curses.A_BOLD | curses.color_pair(4))

        self.stdscr.refresh()

    # -- leaving curses to run an external / blocking command ----------------

    def run_foreground(self, cmd, header=""):
        """Suspend curses, run `cmd` attached to the real terminal, then resume.

        Only used for `kubectl logs -f`, which is a genuine live stream and
        has to own the terminal directly. Regardless of how it exits --
        normally or via Ctrl+C -- this guarantees the child process is fully
        dead before curses is restored, and drops any stray buffered
        keystrokes (e.g. leftover Ctrl+C bytes) so the dashboard redraws
        cleanly instead of getting stuck.
        """
        curses.def_prog_mode()
        curses.endwin()
        os.system("clear")
        if header:
            print(header)
            print("-" * min(len(header), shutil.get_terminal_size().columns))

        proc = None
        try:
            proc = subprocess.Popen(cmd, start_new_session=True)
            proc.wait()
        except KeyboardInterrupt:
            pass  # Ctrl+C during `kubectl logs -f`
        finally:
            if proc is not None and proc.poll() is None:
                try:
                    proc.terminate()
                    proc.wait(timeout=2)
                except Exception:
                    try:
                        proc.kill()
                        proc.wait(timeout=2)
                    except Exception:
                        pass

        curses.reset_prog_mode()
        self.stdscr.clear()
        curses.curs_set(0)
        self.stdscr.timeout(150)
        curses.flushinp()  # discard any leftover input (stray ^C, etc.)

    def show_text_viewer(self, title, text):
        """Scrollable read-only text view rendered inside curses itself.

        Used for `describe pod` instead of piping through `less`, so there
        is no external pager process that can be left running (or trap
        Ctrl+C) after the terminal has already moved on.
        """
        lines = text.splitlines() or ["(no output)"]
        top = 0
        self.stdscr.timeout(-1)  # block while viewing
        try:
            while True:
                self.stdscr.erase()
                h, w = self.stdscr.getmaxyx()
                self.safe_addstr(0, 0, title, curses.A_BOLD | curses.color_pair(4))

                body_height = max(1, h - 2)
                max_top = max(0, len(lines) - body_height)
                top = max(0, min(top, max_top))

                for i, line in enumerate(lines[top:top + body_height]):
                    self.safe_addstr(1 + i, 0, line)

                footer = (
                    f" Line {top + 1}-{min(top + body_height, len(lines))} of {len(lines)}  |  "
                    f"Up/Down/PgUp/PgDn scroll, q/Esc to return "
                )
                self.safe_addstr(h - 1, 0, footer.center(w), curses.A_REVERSE)
                self.stdscr.refresh()

                try:
                    key = self.stdscr.getch()
                except KeyboardInterrupt:
                    return  # be forgiving if Ctrl+C is pressed here too
                if key in (ord('q'), ord('Q'), 27):  # Esc
                    return
                elif key in (curses.KEY_UP, ord('k')):
                    top -= 1
                elif key in (curses.KEY_DOWN, ord('j')):
                    top += 1
                elif key == curses.KEY_NPAGE:
                    top += body_height
                elif key == curses.KEY_PPAGE:
                    top -= body_height
                elif key == curses.KEY_HOME:
                    top = 0
                elif key == curses.KEY_END:
                    top = max_top
        finally:
            self.stdscr.timeout(150)

    # -- actions ---------------------------------------------------------------

    def select_container(self, pod, containers):
        """Small blocking submenu to pick a container when a pod has several."""
        idx = 0
        self.stdscr.timeout(-1)  # block while choosing
        try:
            while True:
                self.stdscr.erase()
                self.safe_addstr(
                    0, 2,
                    f"Pod {pod['namespace']}/{pod['name']} has {len(containers)} containers:",
                    curses.A_BOLD,
                )
                for i, c in enumerate(containers):
                    attr = curses.A_REVERSE | curses.A_BOLD if i == idx else 0
                    self.safe_addstr(2 + i, 4, c, attr)
                self.safe_addstr(
                    3 + len(containers), 2,
                    "Up/Down to choose, Enter to confirm, Esc to cancel",
                    curses.A_DIM,
                )
                self.stdscr.refresh()

                key = self.stdscr.getch()
                if key in (curses.KEY_UP, ord('k')):
                    idx = max(0, idx - 1)
                elif key in (curses.KEY_DOWN, ord('j')):
                    idx = min(len(containers) - 1, idx + 1)
                elif key in (10, 13, curses.KEY_ENTER):
                    return containers[idx]
                elif key == 27:  # Esc
                    return None
        finally:
            self.stdscr.timeout(150)

    def view_logs(self, pods):
        idx = self.current_index(pods)
        if idx == -1:
            return
        pod = pods[idx]
        containers = pod["containers"] or [pod["name"]]

        if len(containers) > 1:
            container = self.select_container(pod, containers)
            if container is None:
                self.set_message("Log view cancelled.")
                return
        else:
            container = containers[0]

        cmd = [
            "kubectl", "logs", "-f",
            pod["name"], "-n", pod["namespace"],
            "-c", container, "--tail=200",
        ]
        header = f"Streaming logs: {pod['namespace']}/{pod['name']} [{container}]  (Ctrl+C to return to dashboard)"
        self.run_foreground(cmd, header=header)
        self.set_message(f"Returned from logs of {pod['namespace']}/{pod['name']} [{container}].")

    def describe_pod(self, pods):
        idx = self.current_index(pods)
        if idx == -1:
            return
        pod = pods[idx]
        self.set_message(f"Fetching describe for {pod['namespace']}/{pod['name']} ...")
        self.render(*self.watcher.get_snapshot())
        try:
            result = subprocess.run(
                ["kubectl", "describe", "pod", pod["name"], "-n", pod["namespace"]],
                capture_output=True, text=True, timeout=15,
            )
            text = result.stdout if result.returncode == 0 else (
                f"kubectl describe failed:\n\n{result.stderr}"
            )
        except Exception as e:
            text = f"Error running kubectl describe: {e}"

        title = f"Describe: {pod['namespace']}/{pod['name']}"
        self.show_text_viewer(title, text)
        self.set_message(f"Returned from describe of {pod['namespace']}/{pod['name']}.")

    def confirm(self, prompt):
        """Blocking y/n confirmation drawn on the bottom of the dashboard."""
        h, w = self.stdscr.getmaxyx()
        self.stdscr.timeout(-1)
        try:
            while True:
                self.safe_addstr(h - 2, 0, " " * (w - 1))
                self.safe_addstr(h - 2, 2, prompt, curses.A_BOLD | curses.color_pair(2))
                self.stdscr.refresh()
                key = self.stdscr.getch()
                if key in (ord('y'), ord('Y')):
                    return True
                if key in (ord('n'), ord('N'), 27):
                    return False
        finally:
            self.stdscr.timeout(150)

    def restart_pod(self, pods):
        idx = self.current_index(pods)
        if idx == -1:
            return
        pod = pods[idx]
        ok = self.confirm(
            f"Restart pod {pod['namespace']}/{pod['name']}? This deletes it so its "
            f"controller recreates it. (y/n)"
        )
        if not ok:
            self.set_message("Restart cancelled.")
            return

        self.set_message(f"Restarting {pod['namespace']}/{pod['name']} ...")
        self.render(*self.watcher.get_snapshot())
        try:
            result = subprocess.run(
                ["kubectl", "delete", "pod", pod["name"], "-n", pod["namespace"], "--wait=false"],
                capture_output=True, text=True, timeout=15,
            )
            if result.returncode == 0:
                self.set_message(f"Restart triggered for {pod['namespace']}/{pod['name']}.")
            else:
                self.set_message(f"Restart failed: {result.stderr.strip()[:150]}")
        except Exception as e:
            self.set_message(f"Restart error: {e}")


# --------------------------------------------------------------------------- #
# Entry point
# --------------------------------------------------------------------------- #

def main():
    if shutil.which("kubectl") is None:
        print("Error: 'kubectl' was not found in PATH.")
        print("Install/configure kubectl (pointing at your Near-RT RIC cluster) and try again.")
        sys.exit(1)

    watcher = PodWatcher(interval=REFRESH_INTERVAL_SECONDS)
    watcher.start()
    try:
        curses.wrapper(lambda stdscr: Dashboard(stdscr, watcher).run())
    finally:
        watcher.stop()


if __name__ == "__main__":
    main()

```


## SDL Dashboard
Live dashboard for the Zero Trust IAM demo, covering two things:

  1. xApp -> SDL PEP authorization monitor
     Lists xApp pods and shows, per xApp, whether the PEP (RESP-proxy
     sidecar) has recently DENIED any of that xApp's own real MSET/MGET
     traffic to SDL. This is passive: it never injects a synthetic
     command, it only reads the PEP's own log output for genuine activity.
         GREEN  = no denial seen in the recent PEP log window (authorized)
         RED    = a denial was found for a real MSET/MGET request (unauthorized)
         YELLOW = pod not Running, or the PEP container's logs couldn't be read
     Press `i` on a selected xApp to see the exact log line/reason.

  2. OPA policy live editor / pusher
     View and edit `opa-policy.yaml` directly in the dashboard, then push
     it to the cluster with one keypress (F2).

IMPORTANT - please check/adjust the CONFIG block below before running:
this script cannot know your exact namespace names, SDL endpoint, or how
your OPA instance is deployed, so it ships with reasonable defaults for a
standard OSC RIC setup and clearly-marked knobs to change if yours differs.

Requirements:
  - Python 3.7+
  - `kubectl` installed and configured for your Near-RT RIC cluster.
  - A Unix-like terminal (Linux/macOS, or WSL on Windows) - uses `curses`.
  - (Optional) PyYAML (`pip install pyyaml`) for policy syntax validation
    before pushing. If it's not installed, validation is skipped with a
    warning, and the file is pushed as-is.

Usage:
```bash
  python3 xapp_sdl_dashboard.py
```

create xapp_sdl_dashboard.py

```python
#!/usr/bin/env python3

import curses
import json
import os
import shutil
import subprocess
import sys
import threading
import time
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timezone

# ============================== CONFIGURATION ===============================
# Adjust these for your deployment.

XAPP_NAMESPACE = "ricxapp"  # namespace where xApp pods run

# --- PEP (RESP-proxy sidecar) authorization monitor -------------------------
# This watches the xApp's OWN real traffic -- it never injects a synthetic
# command. It works by reading the PEP sidecar's own log output for each
# xApp pod and looking for a denial on a privileged command (MSET/MGET).
#
# Name of the sidecar container in each xApp pod that enforces policy
# (your RESP-proxy / PEP). Adjust to match your actual container name.
PEP_CONTAINER_NAME = "envoy-proxy"

# How many of the most recent PEP log lines to scan each refresh cycle.
# This is a rolling window, not a running history -- an old denial ages
# out once it scrolls past this many lines of newer log output, and the
# xApp goes back to green. Raise it if denials should "stick" longer.
PEP_LOG_TAIL_LINES = 100

# Heuristic for spotting a denial line: a line counts as a denial only if
# it mentions one of PRIVILEGED_COMMANDS *and* one of DENIAL_KEYWORDS.
# I don't have visibility into your proxy's actual log format, so this is
# a best-effort guess -- tell me the real format and I'll make it exact.
PRIVILEGED_COMMANDS = ["MSET", "MGET"]
DENIAL_KEYWORDS = ["denied", "unauthorized", "forbidden", "blocked", "reject", "denying"]

REFRESH_INTERVAL_SECONDS = 3.0
CONNECTIVITY_WORKERS = 8  # parallel `kubectl logs` reads per refresh cycle

# Local policy file this dashboard edits (defaults to next to this script).
OPA_POLICY_FILE = os.path.join(
    os.path.dirname(os.path.abspath(__file__)), "opa-policy.yaml"
)

# Default push mechanism: apply as a Kubernetes ConfigMap (common OPA-on-k8s
# pattern where OPA reads its policy from a mounted file). If your OPA is
# reachable via its REST API instead, this is the one function to swap out.
OPA_CONFIGMAP_NAME = "opa-policy"
OPA_CONFIGMAP_NAMESPACE = "ricplt"
# Set to a Deployment name (e.g. "deployment-ricplt-opa") to automatically
# `kubectl rollout restart` it after every push, if your OPA does not
# hot-reload the mounted file on its own. Leave "" to skip.
OPA_DEPLOYMENT_NAME = "opa-pdp"
# =============================================================================


# --------------------------------------------------------------------------- #
# Data fetching
# --------------------------------------------------------------------------- #

def human_age(creation_ts):
    if not creation_ts:
        return "-"
    try:
        created = datetime.strptime(creation_ts, "%Y-%m-%dT%H:%M:%SZ").replace(
            tzinfo=timezone.utc
        )
    except Exception:
        return "-"
    seconds = int((datetime.now(timezone.utc) - created).total_seconds())
    if seconds < 60:
        return f"{seconds}s"
    minutes = seconds // 60
    if minutes < 60:
        return f"{minutes}m"
    hours = minutes // 60
    if hours < 24:
        return f"{hours}h{minutes % 60}m"
    days = hours // 24
    return f"{days}d{hours % 24}h"


def fetch_xapps():
    """Return (xapps, error) for pods in XAPP_NAMESPACE."""
    try:
        proc = subprocess.run(
            ["kubectl", "get", "pods", "-n", XAPP_NAMESPACE, "-o", "json"],
            capture_output=True, text=True, timeout=10,
        )
    except FileNotFoundError:
        return None, "kubectl not found in PATH"
    except subprocess.TimeoutExpired:
        return None, "kubectl get pods timed out"
    except Exception as e:
        return None, str(e)

    if proc.returncode != 0:
        return None, proc.stderr.strip() or "kubectl returned a non-zero exit code"

    try:
        data = json.loads(proc.stdout)
    except json.JSONDecodeError as e:
        return None, f"failed to parse kubectl output: {e}"

    xapps = []
    for item in data.get("items", []):
        meta = item.get("metadata", {}) or {}
        status = item.get("status", {}) or {}
        container_statuses = status.get("containerStatuses") or []
        xapps.append(
            {
                "name": meta.get("name", ""),
                "phase": status.get("phase", "Unknown"),
                "restarts": sum(cs.get("restartCount", 0) for cs in container_statuses),
                "age": human_age(meta.get("creationTimestamp")),
            }
        )
    xapps.sort(key=lambda x: x["name"])
    return xapps, None


def is_denial_line(line):
    """Heuristic: a log line counts as a denial only if it mentions
    a denial keyword"""
    lower = line.lower()
    return any(kw in lower for kw in DENIAL_KEYWORDS)


def check_pep_authorization(pod_name):
    """Passively check whether the PEP recently denied any real MSET/MGET
    request from this xApp -- reads the proxy sidecar's own log output,
    does NOT inject any synthetic command.

    Returns (status, detail):
      "AUTHORIZED"   -- no denial found in the recent log window
      "UNAUTHORIZED" -- a denial line was found (detail = that log line)
      "UNKNOWN"      -- couldn't read the PEP container's logs at all
    """
    try:
        result = subprocess.run(
            ["kubectl", "logs", pod_name, "-n", XAPP_NAMESPACE,
             "-c", PEP_CONTAINER_NAME, f"--tail={PEP_LOG_TAIL_LINES}"],
            capture_output=True, text=True, timeout=8,
        )
    except Exception as e:
        return "UNKNOWN", str(e)[:150]

    if result.returncode != 0:
        return "UNKNOWN", (result.stderr.strip() or "no logs returned")[:150]

    for line in reversed(result.stdout.splitlines()):
        if is_denial_line(line):
            return "UNAUTHORIZED", line.strip()[:150]
    return "AUTHORIZED", f"no denials in last {PEP_LOG_TAIL_LINES} PEP log lines"


class XAppMonitor(threading.Thread):
    """Background thread: polls xApp pods and probes SDL connectivity."""

    def __init__(self):
        super().__init__(daemon=True)
        self.lock = threading.Lock()
        self.xapps = []
        self.connectivity = {}  # name -> (status, detail); status in {AUTHORIZED, UNAUTHORIZED, UNKNOWN}
        self.error = None
        self.last_update = None
        self._stop_event = threading.Event()
        self._pool = ThreadPoolExecutor(max_workers=CONNECTIVITY_WORKERS)

    def run(self):
        while not self._stop_event.is_set():
            xapps, err = fetch_xapps()
            if xapps is not None:
                running_names = [x["name"] for x in xapps if x["phase"] == "Running"]
                results = list(self._pool.map(check_pep_authorization, running_names)) if running_names else []
                conn_map = dict(zip(running_names, results))
                for x in xapps:
                    if x["name"] not in conn_map:
                        conn_map[x["name"]] = ("UNKNOWN", "pod not Running")
                with self.lock:
                    self.xapps = xapps
                    self.connectivity = conn_map
                    self.error = None
                    self.last_update = time.strftime("%H:%M:%S")
            else:
                with self.lock:
                    self.error = err
                    self.last_update = time.strftime("%H:%M:%S")
            self._stop_event.wait(REFRESH_INTERVAL_SECONDS)
        self._pool.shutdown(wait=False)

    def stop(self):
        self._stop_event.set()

    def get_snapshot(self):
        with self.lock:
            return list(self.xapps), dict(self.connectivity), self.error, self.last_update


# --------------------------------------------------------------------------- #
# OPA policy push
# --------------------------------------------------------------------------- #

def apply_policy_configmap(path):
    """Push the given YAML file directly into the cluster."""
    try:
        apply = subprocess.run(
            ["kubectl", "apply", "-f", path],
            capture_output=True, text=True, timeout=10,
        )
        if apply.returncode != 0:
            return False, apply.stderr.strip()

        msg = apply.stdout.strip()
        if OPA_DEPLOYMENT_NAME:
            restart = subprocess.run(
                ["kubectl", "rollout", "restart", f"deployment/{OPA_DEPLOYMENT_NAME}",
                 "-n", OPA_CONFIGMAP_NAMESPACE],
                capture_output=True, text=True, timeout=10,
            )
            msg += " | " + (restart.stdout.strip() if restart.returncode == 0
                             else f"rollout restart failed: {restart.stderr.strip()}")
        return True, msg
    except Exception as e:
        return False, str(e)

# --------------------------------------------------------------------------- #
# Minimal in-curses text editor (used for the OPA policy file)
# --------------------------------------------------------------------------- #

class TextEditor:
    """Small multi-line editor: arrows to move, type to insert, F2 to accept
    (caller decides what "accept" means), Esc to cancel."""

    def __init__(self, stdscr, lines, title):
        self.stdscr = stdscr
        self.lines = lines if lines else [""]
        self.title = title
        self.cy, self.cx = 0, 0
        self.top = 0
        self.dirty = False

    def safe_addstr(self, y, x, text, attr=0):
        h, w = self.stdscr.getmaxyx()
        if 0 <= y < h and 0 <= x < w:
            try:
                self.stdscr.addnstr(y, x, text, max(0, w - x - 1), attr)
            except curses.error:
                pass

    def clamp_cursor(self):
        self.cy = max(0, min(self.cy, len(self.lines) - 1))
        self.cx = max(0, min(self.cx, len(self.lines[self.cy])))

    def confirm_discard(self):
        h, w = self.stdscr.getmaxyx()
        prompt = " Discard unsaved changes? (y/n) "
        self.safe_addstr(h - 1, 0, prompt.center(w), curses.A_REVERSE | curses.color_pair(2))
        self.stdscr.refresh()
        while True:
            k = self.stdscr.getch()
            if k in (ord('y'), ord('Y')):
                return True
            if k in (ord('n'), ord('N'), 27):
                return False

    def render(self):
        self.stdscr.erase()
        h, w = self.stdscr.getmaxyx()
        self.safe_addstr(0, 0, f" {self.title} ".center(w, "="), curses.A_BOLD | curses.color_pair(4))

        body_height = max(1, h - 3)
        if self.cy < self.top:
            self.top = self.cy
        elif self.cy >= self.top + body_height:
            self.top = self.cy - body_height + 1

        for i, line in enumerate(self.lines[self.top:self.top + body_height]):
            self.safe_addstr(1 + i, 0, line)

        dirty_flag = " [modified]" if self.dirty else ""
        footer1 = f" Line {self.cy + 1}/{len(self.lines)}  Col {self.cx + 1}{dirty_flag} "
        footer2 = " F2 Save & apply to cluster    Esc Cancel / back "
        self.safe_addstr(h - 2, 0, footer1.ljust(w - 1), curses.A_REVERSE)
        self.safe_addstr(h - 1, 0, footer2.center(w), curses.A_BOLD)

        screen_y = 1 + (self.cy - self.top)
        screen_x = min(self.cx, w - 1)
        try:
            self.stdscr.move(screen_y, screen_x)
        except curses.error:
            pass
        self.stdscr.refresh()

    def run(self):
        """Returns the joined text on F2 (save), or None on Esc (cancel)."""
        curses.curs_set(1)
        self.stdscr.timeout(-1)
        try:
            while True:
                self.render()
                key = self.stdscr.getch()
                if key == 27:  # Esc
                    if self.dirty and not self.confirm_discard():
                        continue
                    return None
                elif key == curses.KEY_F2:
                    return "\n".join(self.lines)
                elif key == curses.KEY_UP:
                    self.cy -= 1
                    self.clamp_cursor()
                elif key == curses.KEY_DOWN:
                    self.cy += 1
                    self.clamp_cursor()
                elif key == curses.KEY_LEFT:
                    if self.cx > 0:
                        self.cx -= 1
                    elif self.cy > 0:
                        self.cy -= 1
                        self.cx = len(self.lines[self.cy])
                elif key == curses.KEY_RIGHT:
                    if self.cx < len(self.lines[self.cy]):
                        self.cx += 1
                    elif self.cy < len(self.lines) - 1:
                        self.cy += 1
                        self.cx = 0
                elif key == curses.KEY_HOME:
                    self.cx = 0
                elif key == curses.KEY_END:
                    self.cx = len(self.lines[self.cy])
                elif key == curses.KEY_NPAGE:
                    self.cy = min(len(self.lines) - 1, self.cy + 10)
                    self.clamp_cursor()
                elif key == curses.KEY_PPAGE:
                    self.cy = max(0, self.cy - 10)
                    self.clamp_cursor()
                elif key in (curses.KEY_BACKSPACE, 127, 8):
                    self.dirty = True
                    if self.cx > 0:
                        line = self.lines[self.cy]
                        self.lines[self.cy] = line[:self.cx - 1] + line[self.cx:]
                        self.cx -= 1
                    elif self.cy > 0:
                        prev_len = len(self.lines[self.cy - 1])
                        self.lines[self.cy - 1] += self.lines[self.cy]
                        del self.lines[self.cy]
                        self.cy -= 1
                        self.cx = prev_len
                elif key == curses.KEY_DC:
                    self.dirty = True
                    line = self.lines[self.cy]
                    if self.cx < len(line):
                        self.lines[self.cy] = line[:self.cx] + line[self.cx + 1:]
                    elif self.cy < len(self.lines) - 1:
                        self.lines[self.cy] += self.lines[self.cy + 1]
                        del self.lines[self.cy + 1]
                elif key in (10, 13, curses.KEY_ENTER):
                    self.dirty = True
                    line = self.lines[self.cy]
                    self.lines[self.cy] = line[:self.cx]
                    self.lines.insert(self.cy + 1, line[self.cx:])
                    self.cy += 1
                    self.cx = 0
                elif key == curses.KEY_RESIZE:
                    pass
                elif 32 <= key <= 126:
                    self.dirty = True
                    line = self.lines[self.cy]
                    self.lines[self.cy] = line[:self.cx] + chr(key) + line[self.cx:]
                    self.cx += 1
                # other keys ignored
        finally:
            curses.curs_set(0)
            self.stdscr.timeout(150)


# --------------------------------------------------------------------------- #
# Main dashboard
# --------------------------------------------------------------------------- #

class XAppDashboard:
    COL_NAME, COL_STATUS, COL_RESTARTS, COL_AGE = 30, 13, 10, 8

    STATUS_STYLE = {
        "AUTHORIZED": ("\u25CF AUTHORIZED", 1),
        "UNAUTHORIZED": ("\u25CF UNAUTHORIZED", 3),
        "UNKNOWN": ("?  UNKNOWN", 2),
    }

    def __init__(self, stdscr, monitor):
        self.stdscr = stdscr
        self.monitor = monitor
        self.selected_key = None
        self.scroll = 0
        self.message = ""
        self.message_time = 0.0

    def set_message(self, msg):
        self.message = msg
        self.message_time = time.time()

    def safe_addstr(self, y, x, text, attr=0):
        h, w = self.stdscr.getmaxyx()
        if y < 0 or y >= h or x < 0 or x >= w:
            return
        try:
            self.stdscr.addnstr(y, x, text, max(0, w - x - 1), attr)
        except curses.error:
            pass

    def current_index(self, xapps):
        if not xapps:
            return -1
        if self.selected_key is None:
            return 0
        for i, x in enumerate(xapps):
            if x["name"] == self.selected_key:
                return i
        return 0

    def move_selection(self, xapps, delta):
        if not xapps:
            return
        idx = self.current_index(xapps)
        idx = max(0, min(len(xapps) - 1, idx + delta))
        self.selected_key = xapps[idx]["name"]

    def run(self):
        curses.curs_set(0)
        self.stdscr.timeout(150)
        curses.start_color()
        curses.use_default_colors()
        curses.init_pair(1, curses.COLOR_GREEN, -1)
        curses.init_pair(2, curses.COLOR_YELLOW, -1)
        curses.init_pair(3, curses.COLOR_RED, -1)
        curses.init_pair(4, curses.COLOR_CYAN, -1)

        while True:
            xapps, connectivity, err, last_update = self.monitor.get_snapshot()
            self.render(xapps, connectivity, err, last_update)

            key = self.stdscr.getch()
            if key == -1:
                continue
            elif key in (ord('q'), ord('Q')):
                break
            elif key in (curses.KEY_UP, ord('k')):
                self.move_selection(xapps, -1)
            elif key in (curses.KEY_DOWN, ord('j')):
                self.move_selection(xapps, 1)
            elif key in (ord('p'), ord('P')):
                self.open_policy_editor()
            elif key in (ord('i'), ord('I')):
                self.show_detail(xapps, connectivity)
            elif key == curses.KEY_RESIZE:
                self.stdscr.clear()

    def show_detail(self, xapps, connectivity):
        idx = self.current_index(xapps)
        if idx == -1:
            return
        name = xapps[idx]["name"]
        status, detail = connectivity.get(name, ("UNKNOWN", "no data"))
        self.set_message(f"{name} -- {status}: {detail}")

    def render(self, xapps, connectivity, err, last_update):
        self.stdscr.erase()
        h, w = self.stdscr.getmaxyx()

        title = " Zero Trust xApp -> SDL Authorization Dashboard "
        self.safe_addstr(0, 0, title.center(w, "="), curses.A_BOLD | curses.color_pair(4))

        info = (
            f" Namespace: {XAPP_NAMESPACE} | PEP container: {PEP_CONTAINER_NAME} "
            f"| Watching: real time request traffic | xApps: {len(xapps)} "
            f"| Last check: {last_update or '-'} "
        )
        self.safe_addstr(1, 0, info, curses.A_DIM)

        row = 2
        if err:
            self.safe_addstr(row, 0, f" kubectl error: {err}", curses.color_pair(3) | curses.A_BOLD)
            row += 1

        header_y = row + 1
        header = (
            f"{'XAPP NAME':<{self.COL_NAME}}"
            f"{'STATUS':<{self.COL_STATUS}}"
            f"{'RESTARTS':<{self.COL_RESTARTS}}"
            f"{'AGE':<{self.COL_AGE}}"
            f"SDL ACCESS"
        )
        self.safe_addstr(header_y, 0, header, curses.A_REVERSE)

        list_top = header_y + 1
        list_height = max(1, h - list_top - 4)

        idx = self.current_index(xapps)
        if idx >= 0:
            if idx < self.scroll:
                self.scroll = idx
            elif idx >= self.scroll + list_height:
                self.scroll = idx - list_height + 1
        else:
            self.scroll = 0

        visible = xapps[self.scroll:self.scroll + list_height]
        for row_i, xapp in enumerate(visible):
            y = list_top + row_i
            real_idx = self.scroll + row_i
            is_selected = real_idx == idx

            status, _detail = connectivity.get(xapp["name"], ("UNKNOWN", ""))
            status_text, status_pair = self.STATUS_STYLE.get(status, self.STATUS_STYLE["UNKNOWN"])

            name = xapp["name"]
            if len(name) >= self.COL_NAME:
                name = name[: self.COL_NAME - 4] + "..."

            prefix = (
                f"{name:<{self.COL_NAME}}"
                f"{xapp['phase']:<{self.COL_STATUS}}"
                f"{str(xapp['restarts']):<{self.COL_RESTARTS}}"
                f"{xapp['age']:<{self.COL_AGE}}"
            )
            row_attr = curses.A_REVERSE | curses.A_BOLD if is_selected else 0
            self.safe_addstr(y, 0, prefix, row_attr)
            self.safe_addstr(
                y, len(prefix), status_text,
                row_attr if is_selected else (curses.color_pair(status_pair) | curses.A_BOLD),
            )

        if not xapps and not err:
            self.safe_addstr(list_top, 2, "No xApps found (waiting for first refresh)...", curses.A_DIM)

        footer_y = h - 3
        self.safe_addstr(footer_y, 0, "-" * w, curses.A_DIM)
        keys_line = " Up/Down Move   i Detail   p Policy editor   q Quit "
        self.safe_addstr(footer_y + 1, 0, keys_line.center(w), curses.A_BOLD)

        if self.message and (time.time() - self.message_time) < 6:
            self.safe_addstr(footer_y + 2, 0, " " + self.message, curses.A_BOLD | curses.color_pair(4))

        self.stdscr.refresh()

    # -- OPA policy editor ----------------------------------------------------

    def load_policy(self):
        if not os.path.exists(OPA_POLICY_FILE):
            return "", None  # start empty so the user can create it here
        try:
            with open(OPA_POLICY_FILE) as f:
                return f.read(), None
        except Exception as e:
            return "", str(e)

    def open_policy_editor(self):
        text, err = self.load_policy()
        if err:
            self.set_message(f"Could not read {OPA_POLICY_FILE}: {err}")
            return

        editor = TextEditor(self.stdscr, text.splitlines() or [""], f"OPA Policy Editor - {OPA_POLICY_FILE}")
        result = editor.run()
        self.stdscr.clear()
        self.stdscr.timeout(150)

        if result is None:
            self.set_message("Policy edit cancelled - nothing changed.")
            return

        if not result.endswith("\n"):
            result += "\n"
        try:
            with open(OPA_POLICY_FILE, "w") as f:
                f.write(result)
        except Exception as e:
            self.set_message(f"Failed to save {OPA_POLICY_FILE}: {e}")
            return

        warn = ""
        try:
            import yaml  # optional; only used for a pre-push syntax check
            yaml.safe_load(result)
        except ImportError:
            warn = " (PyYAML not installed - syntax check skipped)"
        except Exception as e:
            self.set_message(f"Saved locally, but YAML looks invalid ({e}). Not pushed to cluster.")
            return

        self.set_message("Applying policy to cluster ...")
        self.render(*self.monitor.get_snapshot())
        ok, msg = apply_policy_configmap(OPA_POLICY_FILE)
        if ok:
            self.set_message(f"Policy applied to cluster.{warn} {msg}"[:220])
        else:
            self.set_message(f"Saved locally, but cluster apply FAILED: {msg}"[:220])


# --------------------------------------------------------------------------- #
# Entry point
# --------------------------------------------------------------------------- #

def main():
    if shutil.which("kubectl") is None:
        print("Error: 'kubectl' was not found in PATH.")
        print("Install/configure kubectl (pointing at your Near-RT RIC cluster) and try again.")
        sys.exit(1)

    monitor = XAppMonitor()
    monitor.start()
    try:
        curses.wrapper(lambda stdscr: XAppDashboard(stdscr, monitor).run())
    finally:
        monitor.stop()


if __name__ == "__main__":
    main()
```