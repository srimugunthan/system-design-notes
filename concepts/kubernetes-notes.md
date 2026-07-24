

Here is the reorganized summary of the video **"How To Upgrade Kubernetes Without Downtime"** by KodeKloud:

---

### 1. Scenario & Core Principle

* **The Problem:** Upgrading a 40-node production Kubernetes cluster under heavy $24/7$ traffic without causing downtime.
* **Key Concept:** Running pods **do not** depend on the control plane to keep serving traffic. If the control plane goes offline briefly during an update, existing pods continue running uninterrupted [[00:39](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=39)].
* **Version Compatibility Rule:** Worker nodes can run an older version than the control plane, but **never** a newer version [[00:58](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=58)].

### 2. Upgrade Workflow (Stage-by-Stage)

#### **Stage 1: Control Plane Upgrade**

* Upgrade the control plane (API server, scheduler, control loop) first [[00:52](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=52)].
* Worker nodes continue serving traffic normally while the control plane is updating [[00:52](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=52)].

#### **Stage 2: Worker Node Upgrade (One Node at a Time)**

1. **Cordon the Node:** Mark the node unschedulable so no new pods are placed on it [[01:12](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=72)].
2. **Drain the Node:** Evict existing running pods so Kubernetes can reschedule them onto the remaining 39 nodes [[01:25](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=85)].
3. **Upgrade & Uncordon:** Upgrade the node software, then uncordon it to allow new pods to run on it again [[01:31](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=91)].

#### **Stage 3: Repeat & Scale**

* Repeat the cordoning and draining sequence for the remaining 39 nodes [[02:12](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=132)].
* Once confident in the workflow, nodes can be upgraded in small parallel batches—provided the remaining nodes have sufficient capacity to absorb the displaced workloads [[02:19](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=139)].

---

### 3. Application Protection & Maintenance Best Practices

* **Pod Disruption Budget (PDB):** Use PDBs to set rules (e.g., maintaining at least 2 healthy pods at all times) [[01:56](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=116)]. This forces Kubernetes to wait for replacement pods to become healthy before terminating additional pods on a drained node [[02:04](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=124)].
* **Surge Upgrades (Managed Kubernetes Strategy):** Provision a new node running the updated version *first*, then drain an old node to ensure capacity remains constant [[02:26](https://www.youtube.com/watch?v=jfRxw5-Gdq8&t=146)].

---
