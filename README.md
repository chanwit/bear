# Robot Bear Configuration

A demonstration project for managing Robot Bear resources using Helm, Kustomize, and ConfigHub.

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

### Chart Upgrade Scenario: Engine Bug Fix (v1 → v2)

**Scenario**: The bear's internal engine has a critical bug. The factory releases chart v2 with a fixed engine, but it requires structural changes to the configuration.

#### Current State (v1)

You have two bears running with chart v1:

```bash
# Red bear
helm template my-red ./helm -f helm/values.red.yaml

# Blue bear
helm template my-blue ./helm -f helm/values.blue.yaml
```

#### The Problem

Chart v2 introduces subtle but breaking changes:
- New required field: `ENGINE` (to specify the engine version)
- **Input parameter names changed**: `head` → `headColor`, `body` → `bodyColor`
- **Output fields unchanged**: Still uses `HEAD` and `BODY` (looks compatible!)

The insidious part: The output looks exactly the same, making it appear compatible at first glance.

#### Attempting to Use Old Values with v2

Try to upgrade using your existing red bear configuration with old parameter names:

```bash
helm template my-red ./helm-v2 -f helm-v2/values.red.yaml --set head=green --set body=yellow
```

**Result - Silently ignores your parameters:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v2
kind: Bear
metadata:
  name: my-red-bear
spec:
  ENGINE: v2-fixed
  HEAD: red              # Still red! The --set head=green was ignored
  BODY: red              # Still red! The --set body=yellow was ignored
```

**Why is this confusing?** The output fields (`HEAD`, `BODY`) look identical to v1, suggesting compatibility. But your `--set head=green` is silently ignored because v2's template expects `headColor`, not `head`.

#### Solution: Update to v2 Parameter Names

Use the new v2 parameter names:

```bash
helm template my-red ./helm-v2 -f helm-v2/values.red.yaml --set headColor=green --set bodyColor=yellow
```

**Result - Now it works:**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v2
kind: Bear
metadata:
  name: my-red-bear
spec:
  ENGINE: v2-fixed
  HEAD: green            # Correctly set to green
  BODY: yellow           # Correctly set to yellow
```

#### Migrating Values Files

To fully migrate to v2, you need to update your values files:

**Old v1 values (helm/values.red.yaml):**
```yaml
x: red
head: ""
body: ""
```

**New v2 values (helm-v2/values.red.yaml):**
```yaml
x: red
engine: v2-fixed       # New required field
headColor: ""          # Renamed from 'head'
bodyColor: ""          # Renamed from 'body'
```

#### Chart Upgrade Complexity

This scenario highlights several Helm upgrade challenges:

**Breaking Changes**: Chart upgrades can introduce breaking changes in field names, required fields, or structure. There's no automatic migration - users must manually identify and update all affected values.

**Silent Failures**: Using old parameter names with a new chart doesn't produce errors - the values are simply ignored, leading to unexpected behavior. In our example, setting `--set head=green` silently failed because v2 doesn't recognize the `head` field. The output still shows `HEAD` (making it look compatible), but your color changes are silently discarded. This is particularly insidious because the manifest looks valid and similar to v1, making the issue hard to spot in code reviews or during deployment.

**Discovery Challenge**: Understanding what changed between chart versions requires reading changelogs, comparing templates, or trial and error. The template's new field requirements (`ENGINE`, `headColor`, `bodyColor`) aren't discoverable without inspecting the template file.

**Migration Burden**: Every values file, every CI/CD pipeline, every deployment script must be updated to use new field names. For organizations with many environments, this creates significant migration overhead.

**No Rollback Path**: Once you've updated to v2 values format, you can't easily roll back to v1 without maintaining separate values files for each chart version.

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

### ConfigHub Advantages Over Helm and Kustomize

ConfigHub eliminates the complexity inherent in both Helm's templating and Kustomize's overlay system by treating configuration as queryable structured data. The emergency blue head recall demonstrates this clearly: while Helm requires changing multiple command-line parameters and Kustomize requires switching between different overlay files, ConfigHub solves it with a single query-based command: `cub run set-field --where "Data.spec.HEAD = 'blue'" --path spec.HEAD --value green`. This bulk operation capability means you can fix all affected configurations at once, regardless of how many variants exist, without needing to know their names or locations in advance.

The structured data model provides several key benefits. First, there's no templating language to learn - no Go templates, no value precedence rules, no special syntax. Configuration changes are explicit function calls that operate on specific data paths, making it immediately clear what will change. Second, the query language (`--where`) enables powerful selections across all configurations, something impossible with Helm's parameter-based approach or Kustomize's file-based overlays. You can ask questions like "find all units where HEAD is blue" and operate on them as a set, which is essential for managing configurations at scale.

The upstream/downstream variant system solves a problem that both Helm and Kustomize struggle with: tracking relationships between related configurations. When you update the base bear, ConfigHub knows which variants are downstream and can intelligently merge changes while preserving environment-specific overrides. This is explicit and trackable, unlike Helm's scattered command-line flags or Kustomize's implicit file dependencies. The built-in revision history means every change is versioned automatically - you don't need Git for version control, though you can still use it. This makes rollbacks trivial and audit trails automatic.

For the emergency scenario, ConfigHub's approach is superior in operational efficiency. Helm requires updating every deployment command or values file individually. Kustomize requires either pre-existing overlays or creating new ones on the fly. ConfigHub's query-based approach means the fix is immediate and comprehensive: one command updates all affected configurations, with full revision history, and the ability to preview changes with `--dry-run` before applying. The combination of bulk operations, automatic versioning, and relationship tracking makes ConfigHub particularly powerful for managing large-scale configuration changes across multiple environments.

### How ConfigHub Handles the Engine Upgrade Scenario

Recall the Helm v1→v2 upgrade problem: when the bear's engine has a bug and you need to upgrade the chart, you face silent failures where old parameter names are ignored, requiring manual migration of all values files.

**ConfigHub solves this with its upstream/upgrade mechanism.**

The following is a complete walkthrough demonstrating how to upgrade from v1 to v2 while preserving environment-specific customizations.

#### Setting Up Base and Variants

Starting fresh for this scenario, create a base bear unit (v1 structure):

```bash
cub unit create base-bear bear-v1.yaml
```

**bear-v1.yaml:**
```yaml
apiVersion: v1
kind: Bear
metadata:
  name: robot-bear
spec:
  HEAD: red
  BODY: red
```

Create downstream variants for different environments:

```bash
# Production red bear
cub unit create prod-red-bear --upstream-unit base-bear

# Production blue bear
cub unit create prod-blue-bear --upstream-unit base-bear
cub run set-field --unit prod-blue-bear --path spec.HEAD --value blue
cub run set-field --unit prod-blue-bear --path spec.BODY --value blue
```

#### Engine Bug Discovered - Need to Upgrade to v2

First, render the Helm v2 chart to get the new structure with the fixed engine:

```bash
helm template base-bear ./helm-v2 -f helm-v2/values.red.yaml > bear-v2.yaml
```

**bear-v2.yaml (rendered from Helm v2):**
```yaml
---
# Source: robot-bear/templates/bear.yaml
apiVersion: v2
kind: Bear
metadata:
  name: base-bear-bear
spec:
  ENGINE: v2-fixed
  HEAD: red
  BODY: red
```

Now update the ConfigHub base unit with this new v2 structure:

```bash
cub unit update base-bear bear-v2.yaml
```

The base-bear unit now has the v2 structure with the fixed engine, but all downstream variants are still on v1.

#### Propagate the Upgrade to All Downstream Variants

Now use ConfigHub's upgrade mechanism to merge the changes to all downstream units:

```bash
cub unit update --patch --upgrade \
  --where "UpstreamUnit.Slug = 'base-bear'"
```

**What happens:**
- ConfigHub identifies all units with `base-bear` as their upstream
- It intelligently merges the new `ENGINE` field from upstream
- It **preserves** downstream-specific changes (blue colors in prod-blue-bear)
- All variants now have the v2 structure with their custom colors intact

#### Verify the Upgrade

Check that the blue bear kept its colors while gaining the new engine:

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
  ENGINE: v2-fixed      # New field from upstream
  HEAD: blue            # Preserved downstream override
  BODY: blue            # Preserved downstream override
```

View the diff to see what changed:

```bash
cub unit diff prod-blue-bear --from=-1
```

**Output:**
```diff
--- space/prod-blue-bear/1
+++ space/prod-blue-bear/2
+ apiVersion: v2
- apiVersion: v1
  spec:
+   ENGINE: v2-fixed
    HEAD: blue
    BODY: blue
```

#### Why ConfigHub's Approach Works Better

**Workflow comparison:**

**With Helm alone:**
1. Render v2 chart for red bear: `helm template ... --set headColor=red --set bodyColor=red`
2. Render v2 chart for blue bear: `helm template ... --set headColor=blue --set bodyColor=blue`
3. Update all values files to add `engine: v2-fixed`
4. Update all deployment scripts to use new field names (`headColor`, `bodyColor`)
5. Test each environment individually
6. Deploy to production one environment at a time
7. Deal with silent failures if you missed updating any `--set` flags

**With ConfigHub + Helm:**
1. Render v2 chart once: `helm template base-bear ./helm-v2 ...`
2. Update base unit: `cub unit update base-bear bear-v2.yaml`
3. Upgrade all downstream: `cub unit update --patch --upgrade --where "UpstreamUnit.Slug = 'base-bear'"`
4. Done - all environments upgraded with colors preserved

**Key advantages:**

**Tracked relationships**: ConfigHub explicitly knows that `prod-blue-bear` is downstream from `base-bear`. When you update the base, it can propagate changes intelligently.

**Smart merging**: The `--upgrade` command merges upstream changes while preserving downstream customizations. The ENGINE field is added, but the blue colors remain.

**Single operation**: One command upgrades all downstream units, regardless of how many environments or variants you have.

**Explicit verification**: The diff shows exactly what changed in each unit - no guessing about whether your changes took effect.

**No silent failures**: If there were structural conflicts (e.g., if downstream had modified a field that upstream also changed), ConfigHub would make this explicit in the merge, not silently ignore one side.

**The upstream/downstream concept makes the subtle bug visible**: Here's the critical insight - when you update base-bear from v1 to v2 structure and then try to upgrade downstream units, ConfigHub's diff (shown above) immediately shows you what's changing.

Looking at that diff again, notice what it reveals:
- ✅ **ENGINE field added** - The new required field from v2 is merged in
- ✅ **HEAD: blue preserved** - The downstream customization is kept
- ✅ **BODY: blue preserved** - The downstream customization is kept

If there were an incompatibility (like if v2 changed field names to `head-color`/`body-color`), the diff would show:
```diff
+ HEAD: blue        # Old field still present (potential conflict!)
+ head-color: red   # New field from upstream with default value
```

This makes the problem **immediately visible** - you can see that both the old structure and new structure coexist, revealing the incompatibility. With Helm alone, you'd render the template and see only `head-color: red`, never realizing your `--set head=blue` was silently ignored.

ConfigHub's upstream/downstream tracking creates an audit trail of what changed and what was preserved, making subtle incompatibilities explicit rather than silent.

ConfigHub doesn't replace Helm - it uses Helm to render templates, then manages the variants and upgrades through its upstream/downstream mechanism. This is the key advantage: you render the new chart once, then ConfigHub handles propagating it to all environments while preserving customizations, **and the diff shows you exactly what happened during the merge**.
