# Mission 1 — Mercury · Notebook Playground

> [!NOTE]
> **Did you know?** Mercury has no atmosphere, so temperatures swing between 430 °C (800 °F) during the day and −180 °C (−290 °F) at night — all in the same place. It is also the fastest planet, lapping the Sun in just 88 Earth days.

## Mission Briefing

Mercury is the first planet from the Sun — small, fast, and close to the action. This mission is your first contact with the RHOAI science interface: workbenches. You'll spin up a Jupyter notebook environment, run your first AI-adjacent code, and prove that your workspace survives a restart. Think of it as lighting the engines for the first time. 🛸

## Pre-launch Checklist

- ☀️ Sun mission complete — `DataScienceCluster` is `Ready` and dashboard is accessible
- Team project created in the dashboard (e.g. `mission-control`)
- Storage class available in the cluster (verified in the Sun mission)

## Flight Plan

### Systems Engineering ⚙️

#### 1. Verify the notebook image is pullable

```bash
oc get imagestream -n redhat-ods-applications | grep notebook
```

Confirm standard notebook images are present (e.g. `jupyter-minimal-notebook`, `jupyter-datascience-notebook`).

#### 2. Check resource quotas in the project

```bash
oc get resourcequota -n mission-control
oc get limitrange -n mission-control
```

If no quota exists, the project inherits cluster defaults. Ensure at least **2 CPUs** and **4Gi RAM** are available for a workbench pod.

#### 3. Verify PVC provisioning for home directories

```bash
# A PVC is created automatically when a workbench starts.
# After the Science Crew launches a workbench, verify:
oc get pvc -n mission-control
```

Expected: a PVC in `Bound` state named after the workbench.

### Science Crew 💻

#### 1. Launch a workbench

In the dashboard: **Data Science Projects** → `mission-control` → **Workbenches** → **Create workbench**

Recommended settings:
- **Name**: `mercury-lab`
- **Image**: `Standard Data Science` (latest)
- **Container size**: Small (2 CPU / 4Gi RAM)
- **Cluster storage**: Create new persistent storage (`1Gi` is enough)

Click **Create workbench** and wait for status `Running`.

#### 2. Open JupyterLab and run a notebook

Click **Open** to launch JupyterLab in a new tab.

Create a new notebook (`File → New → Notebook`, select Python 3 kernel) and run:

```python
import pandas as pd
import matplotlib.pyplot as plt

data = pd.DataFrame({
    'planet': ['Mercury', 'Venus', 'Earth', 'Mars'],
    'distance_from_sun_au': [0.39, 0.72, 1.0, 1.52]
})

data.plot(kind='bar', x='planet', y='distance_from_sun_au', legend=False)
plt.title('Distance from the Sun (AU)')
plt.ylabel('AU')
plt.tight_layout()
plt.show()

print("Mission Mercury: notebook online ✅")
```

Expected: a bar chart renders in the output cell and the print statement appears.

#### 3. Install an extra package

In a new cell:

```python
import subprocess
subprocess.run(['pip', 'install', 'httpx'], check=True)

import httpx
print(f"httpx version: {httpx.__version__}")
```

#### 4. Save your work and restart the workbench

Save the notebook (`Ctrl+S`). Go back to the dashboard and **stop** the workbench, then **start** it again.

After restart, open JupyterLab and verify the notebook file is still there.

> [!NOTE]
> Packages installed with `pip` do NOT persist across restarts — only files on the persistent storage do. This is intentional: use `requirements.txt` or a custom notebook image for persistent dependencies.

## GO / NO-GO Checks

- [ ] GO — `oc get notebooks -n mission-control` shows workbench with `Ready` condition
- [ ] GO — `oc get pvc -n mission-control` shows a `Bound` PVC for the workbench
- [ ] GO — Notebook cell with `pandas` plot runs without error
- [ ] GO — After workbench restart, notebook file is present in JupyterLab
- [ ] GO — `oc get pods -n mission-control` shows workbench pod as `Running`

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| Workbench stays in `Starting` | Image pull failure or insufficient quota | `oc describe pod -n mission-control -l app=<workbench-name>` → check `Events` section |
| JupyterLab tab shows "503 Service Unavailable" | Pod started but Jupyter process not yet ready | Wait 60s and refresh; if persistent, check pod logs: `oc logs -n mission-control <pod>` |
| PVC stays in `Pending` | No suitable storage class | `oc get pvc -n mission-control` → check events; verify a default storage class exists |
| Notebook file missing after restart | File saved outside the persistent volume mount | Ensure files are saved under `/opt/app-root/src` (the default home in JupyterLab) |

## Mission Success Criteria

A workbench is running, a Python notebook produces output (including a plot), and after a full stop/start cycle the notebook file is still accessible on persistent storage.

## Bonus Mission Objective

Create a second notebook that uses the `requests` or `httpx` library to call a public REST API (e.g. `https://api.open-meteo.com/v1/forecast?latitude=52.52&longitude=13.41&current_weather=true`) and display the result as a `pandas` DataFrame. This proves your workbench can reach external endpoints — a useful sanity check before you start calling inference APIs in later missions.
