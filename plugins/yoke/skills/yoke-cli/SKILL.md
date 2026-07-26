---
name: yoke-cli
description: Use when the user asks about the "yoke" CLI (github.com/yokecd/yoke) — a Helm-inspired, code-first Kubernetes package deployer. Covers core concepts (flights, wasm), installation, and the main commands (takeoff/up/apply, descent/down/restore, mayday/delete, blackbox/inspect, turbulence/drift/diff, stow/push, sign/verify).
---

# yoke CLI

yoke is a Helm-inspired infrastructure-as-code deployer for Kubernetes. Instead of
templating YAML, you write a normal program (Go is first-class, but any language
that compiles to WASI wasm works) that prints the desired Kubernetes resources as
JSON or YAML to stdout. That compiled program is called a **flight**. yoke embeds
a pure-Go wasm runtime (wazero), runs your flight, and applies the resulting
resources to the cluster — tracking revision history so releases can be diffed,
rolled back, or deleted like a Helm release.

Aviation theme: commands have whimsical primary names with plain aliases
(`takeoff`/`up`/`apply`, `descent`/`down`/`restore`, `mayday`/`delete`,
`blackbox`/`inspect`, `turbulence`/`drift`/`diff`, `stow`/`push`).

## Installation

```bash
go install github.com/yokecd/yoke/cmd/yoke@latest
# or
brew install yoke
```

## Core concept: a flight

A flight is any executable compiled for `GOOS=wasip1 GOARCH=wasm` that writes a
JSON (or YAML) array of Kubernetes resources to stdout. Minimal Go example:

```go
package main

import (
	"encoding/json"
	"os"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func main() {
	dep := appsv1.Deployment{
		TypeMeta:   metav1.TypeMeta{APIVersion: "apps/v1", Kind: "Deployment"},
		ObjectMeta: metav1.ObjectMeta{Name: "sample-app"},
		Spec: appsv1.DeploymentSpec{
			Selector: &metav1.LabelSelector{MatchLabels: map[string]string{"app": "sample-app"}},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{Labels: map[string]string{"app": "sample-app"}},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{{Name: "sample-app", Image: "alpine:latest"}},
				},
			},
		},
	}
	json.NewEncoder(os.Stdout).Encode([]any{dep})
}
```

Compile it to wasm:

```bash
GOOS=wasip1 GOARCH=wasm go build -o main.wasm .
```

Flights can also be plain static resources: if you skip the wasm asset, yoke
reads the desired resources directly from stdin.

## Core commands

### `yoke takeoff` (aliases: `up`, `apply`)

Deploys or updates a release from a flight.

```bash
yoke takeoff my-release ./main.wasm

# gzipped wasm is supported
yoke takeoff my-release ./main.wasm.gz

# remote flight (http(s) or oci:// url)
yoke takeoff my-release https://github.com/my_org/infra/releases/v1.2.3-release.wasm

# pass CLI args through to the flight (after --)
yoke takeoff my-release ./main.wasm -- -env prod

# pass data to the flight over stdin
yoke takeoff my-release ./main.wasm < input.file

# no wasm asset: stdin IS the desired resources (JSON/YAML)
yoke takeoff my-release < resources.json

# preview the diff without applying anything
yoke takeoff -diff-only my-release ./main.wasm
```

Useful flags: `-namespace`, `-create-namespace`, `-dry` (server-side dry run),
`-wait` (wait for resources to become ready, polled via `-poll`), `-out`
(write rendered resources to a directory or `-` for stdout instead of
applying), `-history-cap` (revisions to retain), `-force-conflicts`,
`-force-ownership`.

### `yoke descent` (aliases: `down`, `restore`)

Rolls a release back to a previous revision.

```bash
yoke descent my-release 42
```

### `yoke mayday` (alias: `delete`)

Deletes a release and its entire revision history.

```bash
yoke mayday my-release
```

### `yoke blackbox` (alias: `inspect`)

Inspects releases and revision history.

```bash
yoke blackbox                 # overview of all releases and current revision id
yoke blackbox my-release      # all revisions for a release
yoke blackbox my-release 42   # render resources for a specific revision
yoke blackbox --mapping       # list of resources by release
```

### `yoke turbulence` (aliases: `drift`, `diff`)

Compares the desired release state against actual cluster state (detects drift).

```bash
yoke turbulence my-release                    # diff desired vs cluster state
yoke turbulence -conflict-only=false my-release  # include defaults/managedFields noise
yoke turbulence -fix my-release               # re-apply desired state over any drift
```

### `yoke stow` (alias: `push`)

Pushes a compiled wasm flight to an OCI registry (auth via local `docker login`).

```bash
yoke stow ./main.wasm oci://ghcr.io/org/example
```

### `yoke sign` / `yoke verify`

Sign a wasm module with a private key, and verify it against public keys before
deploying (see `-verify` flag on `takeoff`).

```bash
yoke sign -key private.pem main.wasm
yoke verify -key public.pem main.wasm
```

### `yoke unlatch` (alias: `unlock`)

Clears a stuck release lock left behind by a killed `takeoff`.

```bash
yoke unlatch my-release
```

## Beyond the basics

- `yoke atc` runs the Air Traffic Controller, a Kubernetes operator mode for
  yoke (CRD-driven, continuous reconciliation of flights) — a more advanced
  topic than day-to-day CLI usage.
- `yoke schematics` (alias `meta`) reads/writes embedded metadata on a wasm
  module (e.g. docs, default CLI args).
- Full docs: https://yokecd.github.io/docs
