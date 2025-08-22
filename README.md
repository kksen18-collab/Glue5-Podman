````markdown
# Run AWS Glue 5.0 Locally with Podman + VS Code

This guide shows how to install and run the **AWS Glue 5.0 container image** using **Podman** on Windows, and run Spark scripts locally.  
You’ll be able to develop Glue jobs in **VS Code**, test them with **spark-submit**, and connect with your AWS credentials.

---

## 🛠️ Prerequisites
- [Podman Desktop](https://podman.io/getting-started/installation) installed  
- Podman machine initialized and started:

```powershell
podman machine init
podman machine start
podman info
````

* VS Code with extensions:

  * **Dev Containers** (ms-vscode-remote.remote-containers)
  * **Python** (ms-python.python)
* AWS CLI configured on your host (creates `C:\Users\<you>\.aws\credentials`)

---

## 📂 Project Setup

Create a folder for your Glue project:

```powershell
mkdir C:\Users\<you>\Podman\glue-local
cd C:\Users\<you>\Podman\glue-local
mkdir src
```

Add a sample Spark script at `src/sample.py`:

```python
from pyspark.sql import SparkSession, functions as F

spark = SparkSession.builder.appName("glue5-local").getOrCreate()

df = spark.createDataFrame([(1,"foo"),(2,"bar"),(3,"baz")], "id INT, word STRING")
out = df.withColumn("len", F.length("word"))
out.show(truncate=False)
```

---

## 📥 Pull the Glue 5.0 Image

```powershell
podman pull public.ecr.aws/glue/aws-glue-libs:5
```

---

## ▶️ Run the Glue Container with Bash

From the project root (`C:\Users\<you>\Podman\glue-local`):

```powershell
$env:AWS_PROFILE = "default"   # or your AWS profile name

podman run -it --rm `
  -v "$HOME\.aws:/home/hadoop/.aws:ro" `
  -v "${PWD}:/home/hadoop/workspace" `
  -w /home/hadoop/workspace `
  -e AWS_PROFILE=$env:AWS_PROFILE `
  --entrypoint /bin/bash `
  public.ecr.aws/glue/aws-glue-libs:5
```

This drops you inside the container at `/home/hadoop/workspace`.

---

## 🧪 Test Spark

Inside the container:

```bash
python3 -V
spark-submit --version
spark-submit src/sample.py
```

Expected output:

```text
+---+----+---+
|id |word|len|
+---+----+---+
|1  |foo |3  |
|2  |bar |3  |
|3  |baz |3  |
+---+----+---+
```

---

## 🔁 Run Your Script in One Command

To skip opening Bash:

```powershell
podman run -it --rm `
  -v "$HOME\.aws:/home/hadoop/.aws:ro" `
  -v "${PWD}:/home/hadoop/workspace" `
  -w /home/hadoop/workspace `
  -e AWS_PROFILE=$env:AWS_PROFILE `
  public.ecr.aws/glue/aws-glue-libs:5 `
  spark-submit src/sample.py
```
````markdown
# VS Code Dev Container for AWS Glue 5.0 (Podman on Windows)

This guide sets up a **VS Code Dev Container** that runs your project **inside the AWS Glue 5.0 image** using **Podman** on Windows.  
It also fixes the common “**workspace does not exist**” and “**multiple workspaces**” issues by explicitly controlling the mount.

---

## ✅ What you get
- One clean workspace inside the container (`/workspaces/<your-folder>`).
- Your Windows project folder is bind-mounted into the container.
- Your AWS credentials are mounted at `/home/hadoop/.aws`.
- Spark & Python ready to run (`spark-submit`, `pyspark`, `pytest`).

---

## 🛠️ Prerequisites
- **Podman Desktop** installed, VM started:
  ```powershell
  podman machine init
  podman machine start
  podman info
````

You should see:

```
API forwarding listening on: npipe:////./pipe/docker_engine
```

* **VS Code** with extensions:

  * **Dev Containers** (ms-vscode-remote.remote-containers)
  * **Python** (ms-python.python)
* **AWS CLI** configured on Windows (creates `C:\Users\<you>\.aws\credentials`).
* VS Code setting **Dev Containers: Docker Path** = `podman`.

---

## 📦 Folder layout

Your repo folder (e.g., `C:\Users\<you>\Podman\glue-local`) should look like:

```
glue-local/
├─ src/
│  └─ sample.py
└─ .devcontainer/
   └─ devcontainer.json
```

Example `src/sample.py`:

```python
from pyspark.sql import SparkSession, functions as F

spark = SparkSession.builder.appName("glue5-local").getOrCreate()
df = spark.createDataFrame([(1,"foo"),(2,"bar"),(3,"baz")], "id INT, word STRING")
out = df.withColumn("len", F.length("word"))
out.show(truncate=False)
```

---

## ⚙️ Dev Container configuration

Create **`.devcontainer/devcontainer.json`** with this content:

```json
{
  "name": "Glue 5 on Podman",
  "image": "public.ecr.aws/glue/aws-glue-libs:5",

  "remoteUser": "hadoop",
  "workspaceFolder": "/workspaces/${localWorkspaceFolderBasename}",

  "workspaceMount": "source=${localWorkspaceFolder},target=/workspaces/${localWorkspaceFolderBasename},type=bind,consistency=cached",

  "mounts": [
    "source=${localEnv:USERPROFILE}/.aws,target=/home/hadoop/.aws,type=bind,readonly",
	   "source=${localWorkspaceFolder}/.aws-cache/sso/cache,target=/home/hadoop/.aws/sso/cache,type=bind",
	   "source=${localWorkspaceFolder}/.aws-cache/cli/cache,target=/home/hadoop/.aws/cli/cache,type=bind"
  ],

  "containerEnv": { "AWS_PROFILE": "default" },
  "forwardPorts": [4040],
  "portsAttributes": {
  "4040": { "label": "Spark UI", "onAutoForward": "openBrowser" }
},
  "postCreateCommand": "python3 -m pip install -U pip pytest",

  "customizations": {
    "vscode": {
      "settings": {
        "python.defaultInterpreterPath": "/usr/bin/python3.11",
        "terminal.integrated.defaultProfile.linux": "bash"
      },
      "extensions": ["ms-python.python","ms-vscode-remote.remote-containers"]
    }
  }
}

```

> **Why this fixes “workspace does not exist”**
> Dev Containers normally auto-mounts your folder at `/workspaces/<name>`, but on Podman/Windows that can be skipped depending on context.
> Setting **`workspaceMount`** guarantees the mount exists exactly where **`workspaceFolder`** points.

---

## ▶️ Open in Container (step-by-step)

1. Open **VS Code** in your repo folder (e.g., `glue-local`).
2. Press **F1** → **Dev Containers: Reopen in Container**.
   VS Code will:

   * Pull `public.ecr.aws/glue/aws-glue-libs:5` (if needed)
   * Start a container with your folder mounted at `/workspaces/glue-local`
   * Mount `C:\Users\<you>\.aws` to `/home/hadoop/.aws`
3. When the terminal opens **inside** the container, verify:

   ```bash
   pwd                                  # -> /workspaces/glue-local
   ls -d /workspaces/glue-local         # exists
   ls -la /home/hadoop/.aws             # should show your credentials files
   python3 -V
   spark-submit --version
   spark-submit src/sample.py
   ```

You should see 3 rows with a `len` column in the Spark output.

---

## 🧹 If you saw multiple “workspaces” before

That happens if the project was mounted twice or a `.code-workspace` file added duplicates.

* **Close multi-root workspace**: VS Code → **File → Close Workspace**, then **Open Folder…** and choose the folder (it will open `/workspaces/<repo-name>` inside the container).
* **Remove old dev containers** (optional):

  ```powershell
  podman ps -a --filter "label=devcontainer.local_folder=C:\Users\<you>\Podman\glue-local"
  # podman rm -f <ID1> <ID2> ...
  ```
* **Rebuild clean**: F1 → **Dev Containers: Rebuild Container Without Cache**.

---

## 🔄 Alternative: use `/home/hadoop/workspace` instead

If you prefer that path, swap the two lines in `devcontainer.json`:

```json
"workspaceFolder": "/home/hadoop/workspace",
"workspaceMount": "source=${localWorkspaceFolder},target=/home/hadoop/workspace,type=bind,consistency=cached",
```

Keep **only one** workspace path to avoid duplicates.

---

## 🧪 Optional: One-click run task

Create **`.vscode/tasks.json`** for a quick `spark-submit`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Spark: run sample.py",
      "type": "shell",
      "command": "spark-submit",
      "args": ["${workspaceFolder}/src/sample.py"],
      "group": { "kind": "build", "isDefault": true },
      "problemMatcher": []
    },
    {
      "label": "Spark: run any script…",
      "type": "shell",
      "command": "spark-submit",
      "args": ["${input:scriptPath}"],
      "problemMatcher": [],
      "presentation": { "reveal": "always" }
    }
  ],
  "inputs": [
    {
      "id": "scriptPath",
      "type": "promptString",
      "description": "Path to your PySpark file (inside the dev container)",
      "default": "${workspaceFolder}/src/sample.py"
    }
  ]
}
```

Use **Ctrl+Shift+B** to run `sample.py` or pick **Terminal → Run Task** for any script.

---

## 🛡️ Troubleshooting

### “workspace does not exist”

* Ensure `devcontainer.json` uses **both**:

  ```json
  "workspaceFolder": "/workspaces/${localWorkspaceFolderBasename}",
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspaces/${localWorkspaceFolderBasename},type=bind,consistency=cached"
  ```
* Rebuild: **Dev Containers: Rebuild Container Without Cache**.

### Multiple workspaces showing

* You likely had multiple mounts or a multi-root `.code-workspace`.
  **Close Workspace**, **Open Folder…**, remove stale containers, then rebuild.

### “socket not reachable” (Docker API)

* Podman Desktop → **Settings → Docker API / Docker compatibility** → **Enable**.
* Restart the VM:

  ```powershell
  podman machine stop
  podman machine start
  ```
* In VS Code Settings, **Dev Containers: Docker Path** = `podman`.

### Wrong `DOCKER_HOST`

* Clear any leftover setting:

  ```powershell
  Remove-Item Env:DOCKER_HOST -ErrorAction Ignore
  [Environment]::SetEnvironmentVariable('DOCKER_HOST', $null, 'User')
  [Environment]::SetEnvironmentVariable('DOCKER_HOST', $null, 'Machine')  # admin PS
  ```
* Confirm Podman forwards the named pipe:

  ```powershell
  podman info --format "{{.Host.RemoteSocket.Path}}"
  # expect: npipe:////./pipe/docker_engine
  ```

### Check Glue image runs outside VS Code

```powershell
podman pull public.ecr.aws/glue/aws-glue-libs:5
podman run --rm --entrypoint /bin/sh public.ecr.aws/glue/aws-glue-libs:5 -c "python3 -V && spark-submit --version"
```
Got it 👍 Here’s a **Markdown file** you can drop into your repo (e.g. `SPARK_UI.md`) to explain how to bring up and keep the Spark UI alive inside your Glue 5 + Podman Dev Container.

````markdown
# 🔎 Viewing the Spark UI in Glue 5 Dev Container

When running AWS Glue 5 (Spark 3.5) locally in Podman/VS Code, you can access the **Spark Web UI** to monitor jobs, stages, and executors.

---

## ✅ Enable Spark UI in your script

By default, the Spark UI only starts when an actual action is triggered.  
Use the following example:

```python
from time import sleep
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("ui-check")
    .master("local[*]")
    .config("spark.ui.enabled", "true")
    .config("spark.ui.port", "4040")  # Spark will fall back to 4041/4042 if busy
    .getOrCreate()
)

# Kick off a tiny action to initialize Spark + UI
spark.range(1).count()

# Print the Spark UI URL
print("UI:", spark.sparkContext.uiWebUrl)

# Keep process alive so UI is available in browser
sleep(600)  # 10 minutes
````

---

## 🌐 Accessing the UI

1. In VS Code, ensure your **devcontainer.json** has port forwarding:

```jsonc
"forwardPorts": [4040, 4041, 4042],
"portsAttributes": {
  "4040": { "label": "Spark UI", "onAutoForward": "openBrowser" }
}
```

2. Reopen in container.
3. Run the script above (`python spark_ui_demo.py`).
4. Look at the **VS Code ports tab** → you’ll see `4040` forwarded.
5. Open in browser: [http://localhost:4040](http://localhost:4040)

---

## ⚡ Notes

* If `4040` is in use, Spark will fall back to `4041`, `4042`, etc.
* The UI stays alive only while the Spark context is running → the `sleep(600)` is to keep it open.
* For convenience, put this in a helper script (`spark_ui_demo.py`) you can run anytime.





