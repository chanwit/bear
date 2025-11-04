# Robot Bear Configuration

A demonstration project for managing Robot Bear resources using Helm and Kustomize.

## Helm

### View the Template

To see the Helm template definition:

```bash
cat helm/templates/bear.yaml
```

**Output:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: {{ .Release.Name }}-bear
spec:
  HEAD: {{ .Values.head | default .Values.x }}
  BODY: {{ .Values.body | default .Values.x }}
```

### Render Templates

#### Default Configuration (no-color)

```bash
helm template my-bear ./helm
```

**Result:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v1
kind: Bear
metadata:
  name: my-bear-bear
spec:
  HEAD: no-color
  BODY: no-color
```

#### Red Bear

```bash
helm template my-bear ./helm -f helm/values.red.yaml
```

**Result:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v1
kind: Bear
metadata:
  name: my-bear-bear
spec:
  HEAD: red
  BODY: red
```

#### Blue Bear

```bash
helm template my-bear ./helm -f helm/values.blue.yaml
```

**Result:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v1
kind: Bear
metadata:
  name: my-bear-bear
spec:
  HEAD: blue
  BODY: blue
```

#### Red Bear with Blue Head (Save to File)

Start from Red Bear configuration and override the head to blue, then save to a file:

```bash
helm template my-red ./helm -f helm/values.red.yaml --set head=blue > red_bear.yaml
```

**Result saved to `red_bear.yaml`:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v1
kind: Bear
metadata:
  name: my-red-bear
spec:
  HEAD: blue
  BODY: red
```

#### Blue Bear with Red Body (Save to File)

Start from Blue Bear configuration and override the body to red, then save to a file:

```bash
helm template my-blue ./helm -f helm/values.blue.yaml --set body=red > blue_bear.yaml
```

**Result saved to `blue_bear.yaml`:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v1
kind: Bear
metadata:
  name: my-blue-bear
spec:
  HEAD: blue
  BODY: red
```

### Emergency Configuration Change

**The factory issues an emergency warning: blue heads are faulty! Use green heads instead of blue.**

**How do we achieve this?**

For the Red Bear with Blue Head configuration, simply override the head color from blue to green:

```bash
helm template my-red ./helm -f helm/values.red.yaml --set head=green > red_bear.yaml
```

**Updated result saved to `red_bear.yaml`:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v1
kind: Bear
metadata:
  name: my-red-bear
spec:
  HEAD: green
  BODY: red
```

For the Blue Bear with Red Body configuration, override the head color to green:

```bash
helm template my-blue ./helm -f helm/values.blue.yaml --set body=red --set head=green > blue_bear.yaml
```

**Updated result saved to `blue_bear.yaml`:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v1
kind: Bear
metadata:
  name: my-blue-bear
spec:
  HEAD: green
  BODY: red
```

## Kustomize

### View the Base Configuration

To see the base bear configuration:

```bash
cat kustomize/base/bear.yaml
```

**Output:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: red
  BODY: red
```

### Build Configurations

#### Base Configuration

```bash
kubectl kustomize kustomize/base
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: red
  HEAD: red
```

#### Red Overlay (Blue Head)

The red overlay patches the HEAD to blue:

```bash
kubectl kustomize kustomize/overlays/red
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: red
  HEAD: blue
```

#### With Green Head Component

Using the green-head component:

```bash
kubectl kustomize kustomize/overlays/with-green-head
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: red
  HEAD: green
```

#### Blue Bear

The blue overlay patches both HEAD and BODY to blue:

```bash
kubectl kustomize kustomize/overlays/blue
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: blue
  HEAD: blue
```

#### Blue Bear with Red Body

Starting from Blue Bear and patching the BODY to red:

```bash
kubectl kustomize kustomize/overlays/blue-red-body
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: red
  HEAD: blue
```

### Emergency Configuration Change

**The factory issues an emergency warning: blue heads are faulty! Use green heads instead of blue.**

**How do we achieve this?**

For both the red overlay and blue-red-body overlay configurations that were using blue heads, simply switch to the with-green-head overlay:

```bash
kubectl kustomize kustomize/overlays/with-green-head
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  BODY: red
  HEAD: green
```

This uses the green-head component to patch the HEAD to green, avoiding the faulty blue heads while maintaining the red body.

## Complexity Analysis

### Helm Drawbacks

Helm introduces cognitive complexity through its templating language and value precedence system. Users must understand how `--set` flags override values files, how default values work, and how the template rendering engine processes Go templates. Parameter discovery is challenging - without reading the actual templates, it's difficult to know what values can be overridden or what their valid options are. Debugging issues requires tracing through multiple layers of value precedence (default values → values files → `--set` flags), which can be time-consuming and error-prone. The biggest drawback is reproducibility: command-line overrides create invisible configurations that exist only in deployment scripts or runbooks, making it easy to lose track of what parameters were used in production deployments. This scattered configuration state increases operational risk during incidents or when onboarding new team members.

### Kustomize Drawbacks

Kustomize requires significant upfront investment in creating and organizing overlay and component files. Responding to unexpected configuration changes, like the blue head emergency, can be slow if the needed overlay doesn't already exist - you must create directories, write kustomization files with proper patch syntax, and understand the base/overlay/component hierarchy. The patch syntax itself adds complexity, requiring knowledge of JSON Patch operations (`op: replace`, correct path specifications like `/spec/HEAD`) which are more verbose and error-prone than simple value assignments. The mental model of layered configurations (base → overlays → components) creates indirection - to understand the final output, you must mentally merge multiple files, making it harder to see the complete picture at a glance. For teams that need to make frequent ad-hoc configuration tweaks during development or troubleshooting, the file-creation overhead becomes a significant friction point that slows down iteration cycles.

## ConfigHub

ConfigHub is a configuration management system that treats configuration as structured data without templates, generators, or variable interpolation. It uses **Units** (configuration containers) organized in **Spaces**, with **Variants** created through cloning to manage different environments.

### Setup

Create a space and set it as the default context:

```bash
cub space create robot-bear --set-context
```

### Create Base Bear Configuration

Create a base unit with the red bear configuration:

```bash
cub unit create base-bear bear.yaml
```

**bear.yaml:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: red
  BODY: red
```

### Create Variants (Red Bear with Blue Head)

Clone the base unit to create a variant:

```bash
cub unit create red-blue-head --upstream-unit base-bear
```

Edit the variant to change the HEAD to blue:

```bash
cub unit edit red-blue-head
```

Or use a function to modify specific fields:

```bash
cub run set-field \
  --unit red-blue-head \
  --path spec.HEAD \
  --value blue
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: blue
  BODY: red
```

### Create Blue Bear Variant

Create another variant for a blue bear:

```bash
cub unit create blue-bear --upstream-unit base-bear
```

Update both HEAD and BODY to blue:

```bash
cub run set-field --unit blue-bear --path spec.HEAD --value blue
cub run set-field --unit blue-bear --path spec.BODY --value blue
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: blue
  BODY: blue
```

### Create Blue Bear with Red Body

Clone from blue-bear and modify the BODY:

```bash
cub unit create blue-red-body --upstream-unit blue-bear
cub run set-field --unit blue-red-body --path spec.BODY --value red
```

**Result:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: blue
  BODY: red
```

### Emergency Configuration Change

**The factory issues an emergency warning: blue heads are faulty! Use green heads instead of blue.**

**How do we achieve this?**

Use a function with a filter to update all bears with blue heads:

```bash
cub run set-field \
  --where "Data.spec.HEAD = 'blue'" \
  --path spec.HEAD \
  --value green
```

This command finds all units where `spec.HEAD` is `blue` and changes them to `green` in a single operation.

Alternatively, update specific units individually:

```bash
cub run set-field --unit red-blue-head --path spec.HEAD --value green
cub run set-field --unit blue-red-body --path spec.HEAD --value green
```

### View Changes

See the revision history:

```bash
cub revision list red-blue-head
```

Compare changes:

```bash
cub unit diff red-blue-head --from=-1
```

### Sync Variants with Upstream Changes

If the base unit is updated, propagate changes to downstream variants:

```bash
cub unit update --patch --upgrade \
  --where "UpstreamUnit.Slug = 'base-bear'"
```

This preserves downstream-specific changes while merging upstream updates.

### ConfigHub Advantages Over Helm and Kustomize

ConfigHub eliminates the complexity inherent in both Helm's templating and Kustomize's overlay system by treating configuration as queryable structured data. The emergency blue head recall demonstrates this clearly: while Helm requires changing multiple command-line parameters and Kustomize requires switching between different overlay files, ConfigHub solves it with a single query-based command: `cub run set-field --where "Data.spec.HEAD = 'blue'" --path spec.HEAD --value green`. This bulk operation capability means you can fix all affected configurations at once, regardless of how many variants exist, without needing to know their names or locations in advance.

The structured data model provides several key benefits. First, there's no templating language to learn - no Go templates, no value precedence rules, no special syntax. Configuration changes are explicit function calls that operate on specific data paths, making it immediately clear what will change. Second, the query language (`--where`) enables powerful selections across all configurations, something impossible with Helm's parameter-based approach or Kustomize's file-based overlays. You can ask questions like "find all units where HEAD is blue" and operate on them as a set, which is essential for managing configurations at scale.

The upstream/downstream variant system solves a problem that both Helm and Kustomize struggle with: tracking relationships between related configurations. When you update the base bear, ConfigHub knows which variants are downstream and can intelligently merge changes while preserving environment-specific overrides. This is explicit and trackable, unlike Helm's scattered command-line flags or Kustomize's implicit file dependencies. The built-in revision history means every change is versioned automatically - you don't need Git for version control, though you can still use it. This makes rollbacks trivial and audit trails automatic.

For the emergency scenario, ConfigHub's approach is superior in operational efficiency. Helm requires updating every deployment command or values file individually. Kustomize requires either pre-existing overlays or creating new ones on the fly. ConfigHub's query-based approach means the fix is immediate and comprehensive: one command updates all affected configurations, with full revision history, and the ability to preview changes with `--dry-run` before applying. The combination of bulk operations, automatic versioning, and relationship tracking makes ConfigHub particularly powerful for managing large-scale configuration changes across multiple environments.
