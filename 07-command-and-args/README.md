# Command and Arguments

> Override a container's default startup command and arguments
> directly in the pod spec, without rebuilding the image.

---

## Goal
Understand how `command` and `args` override a container image's
default ENTRYPOINT and CMD.

---

## The Mapping
Docker          Kubernetes

ENTRYPOINT  →   command   (what executable runs)

CMD         →   args      (arguments passed to it)

---

## Files
- `command-demo.yaml` — pod overriding command and args

---

## Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
spec:
  containers:
    - name: democontainer
      image: ubuntu
      command: ["echo"]
      args: ["Hello", "Kubernetes"]
```
runs: echo Hello Kubernetes

---

## Steps
```bash
kubectl apply -f command-demo.yaml

# check the output it printed
kubectl logs command-demo

# should show: Hello Kubernetes
```

---

## The Four Combinations
neither       → image default ENTRYPOINT + CMD

command only  → your command, image CMD ignored

args only     → image ENTRYPOINT, your args replace CMD

both          → fully custom command + args

---

## What I Learned
- command overrides the image's ENTRYPOINT
- args overrides the image's CMD
- if omitted, the image's built-in defaults are used
- command/args run directly, NOT through a shell by default
  → no variable expansion or pipes unless you use sh -c
- useful for one-off tasks, custom flags, keeping containers alive

---

## Gotchas & Mistakes
- tried command: ["echo $HOME"] → $HOME treated as literal text
  fix: wrap in a shell: command: ["sh", "-c", "echo $HOME"]
- (add more as you encounter them)

---

## Cheatsheet

| Field | Overrides | Purpose |
|---|---|---|
| `command` | ENTRYPOINT | what executable to run |
| `args` | CMD | arguments to pass |

| Command | What it does |
|---|---|
| `kubectl apply -f command-demo.yaml` | create the pod |
| `kubectl logs command-demo` | see what the command printed |
| `kubectl describe pod command-demo` | inspect command/args used |
