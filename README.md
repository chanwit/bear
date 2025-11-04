# Robot Bear Configuration

A demonstration project for managing Robot Bear resources, illustrating the trade-offs between Helm, Kustomize, and ConfigHub.

## Helm

### Viewing and Rendering Templates

The Helm chart defines the bear's `HEAD` and `BODY` colors.

**Template Definition (`helm/templates/bear.yaml`):**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: {{ .Release.Name }}-bear
spec:
  HEAD: {{ .Values.head | default .Values.x }}
  BODY: {{ .Values.body | default .Values.x }}
```

You can render templates for different configurations using `values.yaml` files and `--set` overrides.

**Default (no-color):**
```bash
helm template my-bear ./helm
```
```yaml
# Result
apiVersion: v1
kind: Bear
metadata:
  name: my-bear-bear
spec:
  HEAD: no-color
  BODY: no-color
```

**Red Bear:**
```bash
helm template my-bear ./helm -f helm/values.red.yaml
```
```yaml
# Result
apiVersion: v1
kind: Bear
metadata:
  name: my-bear-bear
spec:
  HEAD: red
  BODY: red
```

**Blue Bear:**
```bash
helm template my-bear ./helm -f helm/values.blue.yaml
```
```yaml
# Result
apiVersion: v1
kind: Bear
metadata:
  name: my-bear-bear
spec:
  HEAD: blue
  BODY: blue
```

### Ad-Hoc Configuration

Creating variations is done with command-line overrides.

**Red Bear with Blue Head:**
```bash
helm template my-red ./helm -f helm/values.red.yaml --set head=blue
```
```yaml
# Result
apiVersion: v1
kind: Bear
metadata:
  name: my-red-bear
spec:
  HEAD: blue
  BODY: red
```

### Emergency Change: The "Blue Head Recall"

**Scenario**: An emergency warning is issued: all blue heads are faulty and must be replaced with green ones.

For each bear configuration that uses a blue head, you must manually intervene.

**Fixing a Red Bear with a Blue Head:**
```bash
# Find the original command and change `head=blue` to `head=green`
helm template my-red ./helm -f helm/values.red.yaml --set head=green
```

**Fixing a Blue Bear:**
```bash
# Override the head color to green
helm template my-blue ./helm -f helm/values.blue.yaml --set head=green
```

This manual process must be repeated for every affected release, increasing the risk of human error.

### The Silent Danger of Chart Upgrades (v1 → v2)

**Scenario**: A critical bug is found in the bear's engine. The factory releases chart v2, which fixes the bug but introduces breaking changes to the configuration API.

**The Breaking Changes in v2:**
*   **New Field**: `ENGINE` is now required.
*   **Renamed Parameters**: `head` is now `headColor`, and `body` is now `bodyColor`.
*   **Deceptive Output**: The rendered `spec` still uses `HEAD` and `BODY`, making the change seem backward-compatible.

**The Problem**: If you try to use your old v1 values and overrides with the v2 chart, Helm doesn't warn you. It just silently ignores the old parameters.

```bash
# Attempting to set a green head and yellow body using old v1 parameters
helm template my-red ./helm-v2 -f helm-v2/values.red.yaml --set head=green --set body=yellow
```

**Result: A Silent Failure**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v2
kind: Bear
metadata:
  name: my-red-bear
spec:
  ENGINE: v2-fixed
  HEAD: red              # SILENTLY IGNORED: --set head=green had no effect
  BODY: red              # SILENTLY IGNORED: --set body=yellow had no effect
```
Your intent to change the colors is lost, but you receive no error. This is a common source of configuration drift and deployment errors.

**The Solution**: You must manually update all values files, CI/CD scripts, and command-line overrides to use the new parameter names.

```bash
# Using the correct v2 parameter names
helm template my-red ./helm-v2 -f helm-v2/values.red.yaml --set headColor=green --set bodyColor=yellow
```
```yaml
# Result: It works now
apiVersion: v2
kind: Bear
metadata:
  name: my-red-bear
spec:
  ENGINE: v2-fixed
  HEAD: green            # Correctly set to green
  BODY: yellow           # Correctly set to yellow
```

### Helm: Key Drawbacks

*   **Silent Failures**: As seen in the upgrade scenario, Helm's biggest danger is that it fails silently. Mismatched parameter names are ignored, not flagged, leading to deployments that look successful but are dangerously wrong.
*   **High Cognitive Load**: Users must constantly juggle multiple layers of context: default values, `values.yaml` files, parent chart values, and command-line `--set` overrides. Debugging becomes a tedious process of tracing which value took precedence.
*   **Poor Discoverability**: There is no built-in way to discover what parameters a chart accepts. You must manually read the template files (`{{ .Values.someParameter }}`) to understand your options.
*   **Scattered Configuration**: Using `--set` creates "invisible" configuration that exists only in CI/CD scripts or command history. This makes it nearly impossible to have a single source of truth for what is running in production.
*   **Migration Burden**: Chart upgrades with breaking changes trigger a cascade of manual work. Every values file, script, and pipeline must be found and updated, a process that is both time-consuming and error-prone.

## Kustomize

Kustomize uses a file-based overlay system to manage configuration variants.

### Base and Overlays

A `base` configuration is defined, and `overlays` apply patches to create variants.

**Base Configuration (`kustomize/base/bear.yaml`):**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: red
  BODY: red
```

**Blue Head Overlay (`kustomize/overlays/red/kustomization.yaml`):**
This overlay patches the base to create a bear with a blue head.
```bash
kubectl kustomize kustomize/overlays/red
```
```yaml
# Result
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: red
  HEAD: blue
```

### Emergency Change: The "Blue Head Recall"

**Scenario**: All blue heads must be replaced with green ones.

With Kustomize, the solution is to create a `component` for the green head and apply it to the base.

**Applying the `green-head` Component:**
```bash
kubectl kustomize kustomize/overlays/with-green-head
```
```yaml
# Result
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: red
  HEAD: green
```
This works, but it requires that you have the foresight to structure your overlays and components correctly. If a `green-head` component didn't already exist, you would need to create it first, slowing down your emergency response.

### Kustomize: Key Drawbacks

*   **High Upfront Investment**: Kustomize demands a well-planned directory structure of bases, overlays, and components. Responding to novel, ad-hoc changes is slow because it often requires creating new files and directories.
*   **Verbose and Complex Patching**: JSON/YAML patch syntax is verbose and error-prone. A simple field change requires a multi-line patch structure with a specific `op` and `path`, making it much more complex than a simple value assignment.
*   **Mental Model Overhead**: To understand a final configuration, you must mentally "render" the layers in your head, tracing the base, the applied overlays, and any components. This indirection makes it difficult to see the complete picture at a glance.
*   **Friction for Ad-Hoc Changes**: The file-based model creates significant friction. Need to test a small change? You have to create a new patch file or modify an existing one, which discourages rapid iteration and experimentation.

## ConfigHub: Configuration as Data

ConfigHub treats configuration as structured, queryable data. It avoids templates, generators, and variable interpolation, focusing instead on direct manipulation and inheritance.

### Setup and Base Configuration

Create a `space` for the project and a `unit` for the base configuration.

```bash
# Create a space and set it as the default context
cub space create robot-bear --set-context

# Create a base unit from a file
cub unit create base-bear bear.yaml
```
**`bear.yaml`:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: red
  BODY: red
```

### Creating Variants Through Inheritance

Variants are created by cloning a unit. The new unit inherits the configuration of its `upstream` and can override specific fields.

**Create a Red Bear with a Blue Head:**
```bash
# Clone the base to create a new variant
cub unit create red-blue-head --upstream-unit base-bear

# Set the HEAD field to blue
cub run set-field \
  --unit red-blue-head \
  --path spec.HEAD \
  --value blue
```
**Result (`red-blue-head`):**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: blue # Override
  BODY: red  # Inherited
```

### Emergency Change: The "Blue Head Recall"

**Scenario**: All blue heads must be replaced with green ones.

With ConfigHub, you don't need to know which units are affected. You can find and update them all with a single, powerful command.

```bash
# Find all units where spec.HEAD is 'blue' and set it to 'green'
cub run set-field \
  --where "Data.spec.HEAD = 'blue'" \
  --path spec.HEAD \
  --value green
```

This single operation fixes the issue across all environments and variants simultaneously. Every change is recorded in a revision history, and you can see exactly what changed with `cub unit diff`.

### ConfigHub's Superior Approach to Upgrades

Remember Helm's silent failure during the v1→v2 engine upgrade? ConfigHub solves this elegantly through its upstream/downstream model.

**The Workflow:**
1.  **Render the new v2 chart *once*** to get the target structure.
    ```bash
    helm template base-bear ./helm-v2 > bear-v2.yaml
    ```
2.  **Update the base unit** with the new v2 structure.
    ```bash
    cub unit update base-bear bear-v2.yaml
    ```
3.  **Propagate the upgrade** to all downstream variants.
    ```bash
    cub unit update --patch --upgrade \
      --where "UpstreamUnit.Slug = 'base-bear'"
    ```

**What happens?**
*   ConfigHub identifies all units that inherit from `base-bear`.
*   It intelligently merges the new `ENGINE` field from the v2 structure into every variant.
*   Crucially, it **preserves** all downstream overrides, like the blue colors in `prod-blue-bear`.

**Verification**: Check the upgraded blue bear.
```bash
cub unit get prod-blue-bear --data-only
```
**Result:**
```yaml
apiVersion: v2
kind: Bear
metadata:
  name: robot-bear
spec:
  ENGINE: v2-fixed      # ✅ New field merged from upstream
  HEAD: blue            # ✅ Downstream override preserved
  BODY: blue            # ✅ Downstream override preserved
```
The upgrade is complete, correct, and verifiable across all environments with a single command.

### Why ConfigHub Wins

ConfigHub combines the best of both worlds while avoiding their pitfalls. It provides a structured, repeatable, and scalable approach to configuration management.

*   **No More Silent Failures**: Because ConfigHub operates on structured data, changes are explicit. The `diff` command makes incompatibilities immediately visible, rather than letting them fail silently at runtime.
*   **Configuration as Queryable Data**: The `--where` clause is a game-changer. It lets you manage configurations at scale, performing bulk updates across any number of environments with a single, targeted command. This is impossible with Helm or Kustomize.
*   **Tracked Inheritance**: The upstream/downstream relationship is explicitly tracked. This allows for intelligent merging of upstream changes while preserving environment-specific customizations—a key weakness in other tools.
*   **Built-in Audit Trail**: Every change is automatically versioned. `cub revision list` and `cub unit diff` provide a complete, built-in audit trail without relying on Git commits to understand the history of a configuration.
*   **Drastically Simplified Workflows**: As the upgrade scenario demonstrates, ConfigHub reduces a complex, multi-step, error-prone migration process into a simple, three-step workflow. Render once, update the base, and propagate.

ConfigHub uses tools like Helm for what they do best - templating - and layers on a powerful management plane that makes configuration robust, scalable, and easy to control.