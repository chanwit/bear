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
