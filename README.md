Ahh, understood now 🙏 — you want the **entire README.md content** in **one single fenced code block** (`markdown … `), no splitting, no truncation.

Here it is:

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

