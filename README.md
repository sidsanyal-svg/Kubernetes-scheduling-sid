# Kubernetes Pod Scheduling Lab: Taints, Tolerations, Node Affinity, Pod Affinity & Anti-Affinity

A hands-on lab that teaches **how Kubernetes decides where a pod runs**, using a local multi-node [kind](https://kind.sigs.k8s.io/) cluster. Everything runs on your laptop, no cloud account needed.

To keep things easy to remember, the whole lab uses one analogy: **a city full of pubs**. 🍺

| Kubernetes concept | Pub analogy |
|---|---|
| Taint | The door rule of an invite-only pub |
| Toleration | The invite |
| Node affinity | The hype that makes everyone want to go to one pub |
| Pod affinity | Friends who want to sit together |
| Pod anti-affinity | People who should stay apart, so one bad night doesn't take everyone down |

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Repo layout](#repo-layout)
3. [Part 1: Taints and Tolerations](#part-1-taints-and-tolerations)
4. [Part 2: Node Affinity](#part-2-node-affinity)
5. [Part 3: Pod Affinity and Pod Anti-Affinity](#part-3-pod-affinity-and-pod-anti-affinity)
6. [Summary](#summary)
7. [Cleanup](#cleanup)
8. [What to read next](#what-to-read-next)

---

## Prerequisites

- **Docker** running on your machine
- **[kind](https://kind.sigs.k8s.io/)** installed
- **kubectl** installed

> Each kind "node" is just a Docker container on your laptop, but Kubernetes treats it like a real machine.

---

## Repo layout

All the manifests used in this lab are already in this repo. Apply them in the order described below.

| File | Used in |
|---|---|
| `kind-config.yaml` | Step 1: creates the cluster (1 control plane + 3 workers) |
| `baseline.yaml` | Step 3: 9 plain nginx replicas, no scheduling rules |
| `toleration.yaml` | Step 5: same app, with a toleration added |
| `affinity-required.yaml` | Step 7: toleration + **hard** node affinity |
| `affinity-preferred.yaml` | Step 9: toleration + **soft** node affinity |
| `order.yaml` | Step 12: two plain "order" pods |
| `payment.yaml` | Step 12: "payment" pods with **pod affinity** to order |
| `catalog.yaml` | Step 13: "catalog" pods with **hard pod anti-affinity** |
| `catalog-preferred.yaml` | Step 13: "catalog" pods with **soft pod anti-affinity** |

### Handy command used throughout the lab

This counts how many demo pods are running on each node. We reuse it a lot:

```bash
kubectl get pods -l app=demo-app -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c
```

---

# Part 1: Taints and Tolerations

## The idea

- A **taint** goes on a **node**. It says: *"Do not schedule pods here unless they are allowed."*
- A **toleration** goes on a **pod**. It says: *"I am allowed on nodes with this taint."*

Think of an invite-only pub. By default nobody can walk in, no matter how much money they have. That's the taint. If you have an invite, you can enter. That's the toleration.

⚠️ **Most important point:** an invite does **not** force you to go to that pub. You can still go anywhere else. We'll see this live in Step 5.

## Step 1: Create a multi-node kind cluster

```bash
mkdir k8s-scheduling-lab && cd k8s-scheduling-lab
kind create cluster --name placement-demo --config kind-config.yaml
```

Check all four nodes are ready:

```bash
kubectl get nodes
```

Expected output (versions will differ):

```
NAME                           STATUS   ROLES           AGE   VERSION
placement-demo-control-plane   Ready    control-plane   1m    v1.xx.x
placement-demo-worker          Ready    <none>          1m    v1.xx.x
placement-demo-worker2         Ready    <none>          1m    v1.xx.x
placement-demo-worker3         Ready    <none>          1m    v1.xx.x
```

## Step 2: Look at the default labels and taints

Labels are just text tags attached to a node:

```bash
kubectl get nodes --show-labels
```

You'll see the OS, CPU architecture, and hostname. (On EKS you'd also see things like the instance type.)

Now check the taints:

```bash
kubectl describe nodes | grep -E "^Name:|Taints:"
```

The control plane already has a taint: `node-role.kubernetes.io/control-plane:NoSchedule`. The three workers show `<none>`.

**This is why your application pods never land on the control plane.** Taints aren't some exotic feature. Kubernetes itself uses them in every cluster.

## Step 3: See the default behavior

Apply `baseline.yaml` (9 replicas of nginx) and see where the pods went:

```bash
kubectl apply -f baseline.yaml
kubectl get pods -o wide
```

Then run the [count command](#handy-command-used-throughout-the-lab).

✅ **Expected:** pods are spread roughly evenly across the three workers. Nothing goes to the control plane because of its taint. Exact numbers can vary slightly, and that's fine.

## Step 4: Add a taint to a worker node

Let's reserve `worker3` for payment workloads only:

```bash
kubectl taint nodes placement-demo-worker3 dedicated=payment:NoSchedule
```

A taint has three parts:

| Part | Value here | Meaning |
|---|---|---|
| Key | `dedicated` | A name we chose |
| Value | `payment` | A name we chose |
| Effect | `NoSchedule` | The actual rule |

### The three effects

| Effect | What it does |
|---|---|
| `NoSchedule` | New pods without a matching toleration will **not** be scheduled on this node |
| `PreferNoSchedule` | The scheduler *tries* to avoid this node, but it's not strict |
| `NoExecute` | New pods are blocked **and** pods already running here are **evicted** |

### A small trap

Run the count command again. You'll **still see pods on worker3**. `NoSchedule` only affects *new* pods, and the existing ones stay where they are.

To see the taint working, force new pods to be created:

```bash
kubectl rollout restart deployment demo-app
```

Wait a few seconds and count again. ✅ Now nothing runs on `worker3`. All pods are on `worker` and `worker2`.

Scale up and count again, and still nothing lands on `worker3`:

```bash
kubectl scale deployment demo-app --replicas=15
```

## Step 5: Add a toleration to the pods

Apply `toleration.yaml`. It's the same deployment (12 replicas) with a `tolerations` section added. The toleration must **match the taint** (same key, value, and effect).

```bash
kubectl apply -f toleration.yaml
```

Run the count command.

✅ **Expected:** you now see pods on `worker3`... **but also** on `worker` and `worker2`.

> **A toleration is only a pass. It is not a direction.** The pods are *allowed* on `worker3`, but nothing tells them to *go* there. Making pods go to a specific node is the job of **node affinity** (Part 2).

## What to remember

- A taint sits on a **node** and keeps pods away.
- A toleration sits on a **pod** and lets it ignore a taint.
- A toleration *allows* a pod on a node. It does **not send** the pod there.
- `NoSchedule` does not remove already-running pods. `NoExecute` does.
- Taints and tolerations are how you protect special nodes like GPU nodes or payment-only nodes.

## 🧪 Practice it yourself

See `NoExecute` in action. Taint `worker2` and watch the pods leave:

```bash
kubectl taint nodes placement-demo-worker2 maintenance=true:NoExecute
kubectl get pods -o wide
```

Your pods have no toleration for `maintenance`, so the ones on `worker2` are evicted and recreated elsewhere. When done, remove the taint (the trailing `-` removes it):

```bash
kubectl taint nodes placement-demo-worker2 maintenance=true:NoExecute-
```

---

# Part 2: Node Affinity

## The idea

Right now: `worker3` is an invite-only pub (taint), and our pods carry an invite (toleration). But nobody is telling them to go there.

Let's make that pub popular. We do it with a **label on the node** and a **node affinity rule on the pod**. *Affinity* simply means attraction: the pod is attracted to nodes carrying a certain label.

The pub is still invite-only, so a pod needs **both** things to get in: the invite (toleration) **and** the desire to go (affinity).

## Step 6: Label the node

```bash
kubectl label nodes placement-demo-worker3 hype=true
kubectl get nodes -l hype=true
```

Only `worker3` should show up. A label alone does nothing. It neither blocks nor attracts pods. It's just a tag until a pod says it cares about it.

## Step 7: Add a hard rule (`required`)

Apply `affinity-required.yaml` (toleration + a required node affinity on `hype=true`):

```bash
kubectl apply -f affinity-required.yaml
kubectl rollout status deployment demo-app
```

Run the count command.

✅ **Expected:** **all 12 pods are on `worker3`.** Compare with Step 5, where pods were spread everywhere. *The toleration let them in. The affinity sent them there.*

### Understanding the new YAML fields

| Field | Meaning |
|---|---|
| `requiredDuringScheduling...` | A **hard rule**. The pod *must* go to a matching node |
| `matchExpressions` | Find nodes where label `hype` has value `true` |
| `operator: In` | "The value is in this list". A pod can use *any* node with the label, even if many nodes share it |
| `...IgnoredDuringExecution` | The rule is only checked when the pod is **being placed**. If you remove the label later, running pods stay where they are |

> The value `"true"` is in quotes because Kubernetes expects a string.

## Step 8: What happens when the pub is full

With a **hard rule**, everyone insists on that one pub. When it's full, they wait outside, even if other pubs are empty. In Kubernetes, waiting pods sit in the **Pending** state.

Simulate a full pub. `cordon` closes a node's doors so no new pods can be placed on it:

```bash
kubectl cordon placement-demo-worker3
kubectl delete pods -l app=demo-app
kubectl get pods -o wide
```

❌ **Expected:** all 12 new pods are **Pending**. Pick one and describe it:

```bash
kubectl describe pod <pod-name>
```

Read the **Events** at the bottom. It says no node is available: one node is unschedulable and the others don't match the pod's node affinity rule. (Exact wording depends on your Kubernetes version.)

**When is this what you want?** If a pod needs a GPU, it should wait for a GPU node rather than run somewhere it can't work. But be aware: **a hard rule can leave pods stuck in Pending if there's no room.**

## Step 9: Switch to a soft rule (`preferred`)

A **soft rule** says: *"Try to go to this pub. If you can't get in, go somewhere else."*

Apply `affinity-preferred.yaml`. It's the same deployment, but uses `preferred...` instead of `required...`, and adds a `weight` (1 to 100) that tells the scheduler how strongly the pod prefers that node. Weight matters more when you have several preferences.

```bash
kubectl apply -f affinity-preferred.yaml
kubectl rollout status deployment demo-app
kubectl get pods -o wide
```

Run the count command. ✅ **Expected:** pods are running on `worker` and `worker2`. **Nothing is Pending.** They wanted the hyped pub, it was full, so they went elsewhere.

## Step 10: Open the doors again

```bash
kubectl uncordon placement-demo-worker3
kubectl rollout restart deployment demo-app
kubectl get pods -o wide
```

Wait for the rollout and count again. ✅ **Expected:** *most* pods land on `worker3`. Not necessarily all. A soft rule is a **preference, not a promise**, and the scheduler also weighs other things, like spreading pods out.

## Hard rule vs. soft rule

| | Hard rule (`required`) | Soft rule (`preferred`) |
|---|---|---|
| Behavior | Pod **must** go to a matching node | Pod **tries** to go there, falls back otherwise |
| Risk | Pods can get stuck **Pending** | Pods may land somewhere less ideal |
| Use when | The pod can't work anywhere else (e.g. a **GPU** workload) | You have a preference but can live without it (e.g. heavy pods preferring big nodes, falling back to small nodes while big ones start up) |
| Protects | **Correctness** | **Availability** |

## What to remember

- A **label** is a tag on a node. **Node affinity** is a pod saying it's attracted to nodes with that tag.
- Taint alone → keeps pods out. Toleration alone → lets them in. Affinity alone → pulls them in.
- **To build a truly dedicated node you need all three:** taint (keep others out) + toleration (let your pods in) + affinity (send them there).
- `required` = hard rule, can leave pods Pending. `preferred` = soft rule, falls back to other nodes.
- Both are checked only at scheduling time, because of `IgnoredDuringExecution`.

## 🧪 Practice it yourself

1. **Affinity alone isn't enough.** Open `affinity-required.yaml`, delete the whole `tolerations` block, and apply it. Pods want the pub but have no invite, so they stay **Pending**.
2. **`IgnoredDuringExecution` in action.** Put the toleration back and apply again. Once all pods are on `worker3`, remove the label:
   ```bash
   kubectl label nodes placement-demo-worker3 hype-
   ```
   The running pods **do not move**.

---

# Part 3: Pod Affinity and Pod Anti-Affinity

## The idea

So far we controlled placement based on **nodes**. Now we control it based on **other pods**.

Some people talk all day and want to sit in the same pub. Others can't stand each other and should go to different pubs. Pods are the same:

- **Pod affinity:** *"Put me on the same node as pods with this label."*
- **Pod anti-affinity:** *"Keep me away from pods with this label."*

### Why would you want this?

- **Affinity (latency and cost):** In a microservices app, an order service and a payment service may talk constantly. On the same node, their traffic needs no extra network hops, saving milliseconds. If they're in different zones, you also pay a small data-transfer cost per request. One request costs almost nothing, but with 100,000 users it adds up to real money.
- **Anti-affinity (resilience):** You don't want all copies of the same service on one node. If that node dies, every copy dies with it.

*(If you've watched Bigg Boss: the friends who chat all day share a room, and the people who fight get separate rooms.)*

## Step 11: Reset the cluster

Remove the demo app, the taint, and the label from before so all three workers are open:

```bash
kubectl delete deployment demo-app
kubectl taint nodes placement-demo-worker3 dedicated=payment:NoSchedule-
kubectl label nodes placement-demo-worker3 hype-
```

> If the last command says the label was not found, that's fine. You already removed it during practice.

From here on, use this command to see where each pod is running:

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName --sort-by=.spec.nodeName
```

## Step 12: Pod affinity, keep payment next to order

- `order.yaml` runs two plain "order" pods with no rules.
- `payment.yaml` says payment pods **must run on a node where an order pod is already running**.

⏱️ **Order matters.** Pod affinity looks at pods that *already exist*, so order pods must be running before payment pods are scheduled:

```bash
kubectl apply -f order.yaml
kubectl rollout status deployment order
kubectl apply -f payment.yaml
kubectl rollout status deployment payment
```

Check where everything landed:

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName --sort-by=.spec.nodeName
```

✅ **Expected:** every payment pod is on a node that also has an order pod. (Both order pods might land on the same node, so the payment pods will be there too. That's a small risk, and Step 13 shows how to deal with it.)

### Understanding the new YAML fields

| Field | Meaning |
|---|---|
| `labelSelector` | Which **pods** to look at (here: pods labeled `app=order`). This is the big difference from node affinity, which looks at labels on *nodes* |
| `topologyKey` | What "together" means. `kubernetes.io/hostname` = **the same node** (each node has a unique hostname) |
| `required` / `preferred` | Work exactly like in node affinity: hard rule vs. soft rule |

> **On EKS**, you can use `topology.kubernetes.io/zone` as the topology key. Then "together" means "in the same availability zone". Kind has no zones, so we can't try that here.

## Step 13: Pod anti-affinity, keep copies apart

`catalog.yaml` runs two catalog pods, and each one **refuses to share a node with another catalog pod**. Notice the pod points at *its own label*: *"Don't put me on a node that already has another catalog pod."*

```bash
kubectl apply -f catalog.yaml
kubectl rollout status deployment catalog
kubectl get pods -l app=catalog -o wide
```

✅ **Expected:** the two catalog pods are on two different nodes. Always.

### Now push it

We only have three workers, so ask for four copies:

```bash
kubectl scale deployment catalog --replicas=4
kubectl get pods -l app=catalog
```

❌ **Expected:** three pods Running, **one Pending**. Describe the pending pod and read the events: the scheduler says no node satisfies the anti-affinity rule.

Same lesson as node affinity: **a hard rule can leave pods Pending when there's no room.**

### The fix: a soft rule

Apply `catalog-preferred.yaml` (4 replicas, soft anti-affinity):

```bash
kubectl apply -f catalog-preferred.yaml
kubectl get pods -l app=catalog -o wide
```

✅ **Expected:** all four pods run. The scheduler still *tries* to spread them but won't leave any pending.

> In a soft pod rule, the selector sits inside `podAffinityTerm`, next to the `weight`. That's the extra layer compared to the hard version.

## Step 14: Why spreading matters, kill a node

Go back to two catalog pods with the hard rule:

```bash
kubectl apply -f catalog.yaml
kubectl rollout status deployment catalog
kubectl get pods -l app=catalog -o wide
```

Pick one of the two nodes from the `NODE` column. In kind, every node is a Docker container, so we can simply stop it (replace the name with your node):

```bash
docker stop placement-demo-worker2
```

Wait about 2-3 minutes, then check:

```bash
kubectl get nodes
kubectl get pods -l app=catalog -o wide
```

✅ **Expected:** the stopped node shows `NotReady`. The catalog pod on it is no longer ready, **but the other catalog pod is on a different node and is still `1/1 Running`. Your service is still answering requests.**

Without anti-affinity, both pods could have been on the same node, and both would have gone down together, giving your users errors until Kubernetes recovers.

**One more thing to know:** by default, Kubernetes waits about **five minutes** before evicting pods from a dead node and recreating them elsewhere. On a real cluster, a node autoscaler like **Karpenter** or **Cluster Autoscaler** would also bring up a replacement node. Kind has no autoscaler, so nothing new appears here. That's exactly why we spread pods: **the healthy copy keeps serving while the cluster recovers.**

Bring the node back when you're done:

```bash
docker start placement-demo-worker2
```

Give it a minute and check `kubectl get nodes`. If the node doesn't come back cleanly, delete the cluster and recreate it (takes about a minute).

## When to use which

- **Pod affinity:** two services talk a lot and you want low latency and lower data-transfer cost. Keep it to pairs that *really* need it. The more you tie pods together, the harder it is for the scheduler to place them.
- **Pod anti-affinity:** you have several copies of one service and want them to survive a node failure. On EKS, use the **zone** topology key to survive a whole zone failure.
- **Be careful with hard rules.** Too many `required` rules can leave pods Pending, especially on a small cluster.

## What to remember

- Pod affinity/anti-affinity look at labels on **other pods**. Node affinity looks at labels on **nodes**.
- `topologyKey` decides what "together" means: `hostname` = same node, `zone` = same availability zone.
- Pod affinity needs the target pods to **exist already**, so start them first.
- Anti-affinity against your **own label** is the simplest way to spread the copies of one service.

## 🧪 Practice it yourself

1. **Affinity with nothing to sit with.** Scale order to zero, then delete the payment pods:
   ```bash
   kubectl scale deployment order --replicas=0
   kubectl delete pods -l app=payment
   ```
   The new payment pods stay **Pending** because there's no order pod to sit with. Scale order back to 2 and watch them run.
2. **Best of both worlds.** Add a `podAntiAffinity` block against their own labels to both `order.yaml` and `payment.yaml`, delete both deployments, and apply `order.yaml` first. You should end up with one order pod and one payment pod on each of two nodes: **the pair stays together, and the copies stay apart.**

---

# Summary

Back to the pub one last time:

| Tool | Sits on | What it does | Pub analogy |
|---|---|---|---|
| **Taint** | Node | Keeps pods away unless they're allowed | Door rule of an invite-only pub |
| **Toleration** | Pod | Allows a pod onto a tainted node | The invite (doesn't force you to go) |
| **Node affinity** | Pod | Attracts a pod to nodes with certain labels | The hype that makes everyone want that pub |
| **Pod affinity** | Pod | Places a pod near other pods | Friends who sit together |
| **Pod anti-affinity** | Pod | Keeps a pod away from other pods | People who should stay apart |

## Real-world combinations

In a real project you'll rarely use these one at a time:

- A **GPU node pool** uses a taint + a toleration + a required node affinity together.
- A **microservices app** uses pod affinity for chatty pairs and anti-affinity for high availability.

You design these rules once, when planning your deployment, and then leave them alone. **You don't need to memorize the syntax.** What matters is knowing which tool solves which problem.

---

# What to read next

**Topology spread constraints** let you spread pods evenly across nodes and zones *by default*, without writing anti-affinity rules for every app. A natural next step after this lab.
