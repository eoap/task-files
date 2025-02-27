# Taskfile for Building and Testing CWL Workflows

This repository provides a **Taskfile** to automate the **build** and **test** processes for CWL tools and workflows. The Taskfile supports **both local and in-cluster execution**.

---

## **Available Tasks**

### **Building CWL Tool Containers**

#### **Build Locally with Docker/Podman**

```sh
task build-local
```

* Uses Docker or Podman to build CWL tool containers. 
* Pushes images to a container registry if configured.

#### **Build In-Cluster with Skaffold & Kaniko**

```sh
task build-cluster
```

* Uses **Skaffold and Kaniko** to build images inside Kubernetes. 
* Generates a `skaffold-auto.yaml` configuration dynamically. 
* Pushes images to the specified container registry.

#### **Run Both Local and In-Cluster Builds**

```sh
task build
```

* Runs `build-local` and `build-cluster` sequentially.
* Updates CWL workflows with the latest image references.

---

### **Testing CWL Workflows and Tools**

#### **Run All Workflow Tests**

```sh
task test
```

* Executes tests for all defined CWL workflows. 
* Automatically selects **Calrissian (in-cluster) or CWLTool (local)** based on configuration. 
* Logs test outputs and execution reports.

#### **Run Tests for a Specific Tool**

```sh
task test-tool VAR=<tool_name>
```

* Runs **only the tests** for a specific CWL tool. 
* Uses the configured execution engine (local or cluster).

---

## **How It Works**

### **Environment Detection**

The Taskfile determines whether to run builds and tests **locally** or **in-cluster** based on the configured execution engine.

### **Dynamic Build Configuration**

- For in-cluster builds, **Skaffold configuration** is generated dynamically.
- **Tools and workflows are extracted** from the configuration and processed.

### **Automated Testing**

- Runs **workflow and tool tests automatically**, applying the correct **execution engine** (CWLTool or Calrissian).
- Stores **logs and output files** in the defined locations.

---

## **Example Usage**

### **Run Everything: Build and Test**

```sh
task build test
```

* Builds CWL tool containers **locally and in-cluster**. 

*  Updates CWL workflows with **the latest image references**. 

* Runs **all tests for CWL workflows and tools**.

---

## **Troubleshooting**

### **How Do I Change the Execution Engine?**

Modify the build and test settings to switch between **local execution** (Docker/Podman) and **in-cluster execution** (Kubernetes + Calrissian).

### **How Do I Debug Build Failures?**

- Run `task prepare-kaniko` to inspect the Skaffold configuration.
- Run `skaffold build -f skaffold-auto.yaml -v=debug` for more details.

### **How Do I Debug Test Failures?**

- Check log files in the configured output directories.
- Ensure that **CWL inputs** and **dependencies** are correct.


