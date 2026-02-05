---
title: "How I Built Graft: An Overlay Engine for Terraform Modules"
date: 2026-02-01 08:00:00 +0800
categories: [Tools]
tags: [graft, terraform, overlay, modules]
---

There's a [Terraform GitHub issue](https://github.com/hashicorp/terraform/issues/27360) that's been open for years: people want to customize modules without forking them. Add a lifecycle block. Tweak a tag. Simple stuff.

I understand why Terraform doesn't support this natively—modules are supposed to be black boxes, and breaking the encapsulation is not ideal. But in practice, modules often need tweaks.

I built Graft to solve this. It patches Terraform modules in place—no forks, no merge conflicts. 

And honestly, it's a middleware for something bigger I'm working on. But I'll save that for the next post. :)

## The Idea

The goal: use declarative Terraform blocks to describe modifications to existing modules. It should:

- Modify multi-layer (nested) modules
- Work easily with existing modules
- Stay compatible when modules update

So I can define a graft manifest like this:

```hcl
module "network" {
  override {
    # patches to modify the existing module
  }

  module "subnet" {
    override {
      # patches to modify the nested module
    }
  }
}
```

The nested structure mirrors the module hierarchy. This makes it easy to locate exactly which blocks you want to modify—just navigate down the tree.

## First Attempt: Override Files

My first idea was to use Terraform's native override mechanism. If you create `override.tf`, it merges with your main config. ([Official docs](https://developer.hashicorp.com/terraform/language/files/override))

But override files have serious limitations:

1. You can't add new blocks—only modify existing ones
2. You can't delete blocks or attributes

Not enough.

## Second Attempt: Enhanced Override Files

Since the graft manifest is processed *before* Terraform runs, I have more control than native overrides.

**Adding new blocks** was easy: check the source code, then generate a new file `_graft_add.tf` in the module directory.

**Deleting things** required a new approach. The implementation wasn't hard—just parse the manifest and remove matching blocks from the source files. But the design was tricky: how do you express "delete this" in a way that feels native to Terraform?

I introduced a special `_graft` block:

```hcl
resource "azurerm_network_security_rule" "allow_all" {
  _graft {
    remove = ["self"]  # Delete the entire resource
  }
}

resource "azurerm_virtual_network" "vnet" {
  _graft {
    remove = ["dns_servers", "tags"]  # Delete specific attributes
  }
}
```

It looks like regular HCL. It nests inside the resource block you're targeting. It follows Terraform's declarative style. That's what I wanted—something that feels like it *belongs* in Terraform, even though Terraform itself can't do this.

## Referencing Original Values

While testing the override strategy, I ran into an interesting problem with `count` and `for_each` resources.

Say a module creates multiple subnets with `for_each`, and I want to modify just one of them. I can target a specific key:

```hcl
resource "azurerm_subnet" "main" {
  for_each = var.subnets
  
  # Only modify subnet1
  service_endpoints = each.key == "subnet1" ? ["Microsoft.Storage"] : ???
}
```

But what goes in the `???`? I need the *original* value to avoid affecting other subnets. Without knowing what the module originally set, I'd have to hardcode it—or worse, accidentally break the other subnets.

This is where `graft.source` came from. It references the original value—no matter how complicated the expression is. I don't need to look it up in the module source code.

```hcl
service_endpoints = each.key == "subnet1" ? ["Microsoft.Storage"] : graft.source
```

This also solves another frustration with Terraform's native override files: they use **shallow merge** for attributes. If you want to add one tag, you can't—your override *replaces* the entire `tags` map, wiping out the module's defaults.

With `graft.source`, you can actually merge:

```hcl
tags = merge(graft.source, {
  "Owner" = "Platform Team"
})
```

During patching, `graft.source` gets replaced with the actual original expression. You get true merging—and you don't need to know what the original value was.

## The Linker Problem

Now I had patching working. But how do I make Terraform *use* the patched modules?

My first idea: use an override file to redirect the module source to a local patched copy.

```hcl
# file: _graft_override.tf
# What I tried to generate
module "network" {
  source = "./.graft/patched-network"
}
```

It failed immediately.

**You can't override `source` when there's a `version` constraint:**

```hcl
# Original main.tf
module "network" {
  source  = "Azure/network/azurerm"
  version = "5.3.0"  # ← This kills the override
}
```

Terraform throws: *"Cannot apply a version constraint to module 'network' because it has a relative local path."*

And you can't "unset" the version—override files can only add or modify, never delete.

Dead end.

## The Breakthrough: Hijacking modules.json

I started digging into how Terraform actually resolves modules.

When you run `terraform init`, Terraform downloads modules and records their locations in `.terraform/modules/modules.json`:

```json
{
  "Modules": [
    {
      "Key": "network",
      "Source": "registry.terraform.io/Azure/network/azurerm",
      "Version": "5.3.0",
      "Dir": ".terraform/modules/network"
    }
  ]
}
```

What if I just changed where `Dir` points? I tried it manually—edited `modules.json`, pointed `Dir` to a local folder with patched code.

It worked. Terraform loaded my patched module while believing it was using the official registry version. No errors. No need to modify `main.tf`.

I called this the **Linker Strategy**—like how linkers resolve symbols to addresses, Graft resolves modules to patched directories.

## The Scaffold Command

One thing bothered me. Graft's whole point is that you shouldn't need to understand a module's internals—just declare what you want to change.

But when I actually used it, I kept opening module source files anyway. Which nested module contains that resource? What's the hierarchy? Even as the author, I couldn't write a manifest without digging through `.terraform/modules`.

So I added `graft scaffold`. It scans your `.terraform/modules` directory and generates a starter manifest with the full module tree:

```bash
$ graft scaffold

[+] Discovering modules in .terraform/modules...
root
├── network (registry.terraform.io/Azure/network/azurerm, 5.3.0)
│   └── [3 resources]
└── compute (registry.terraform.io/Azure/compute/azurerm, 5.3.0)
    ├── [18 resources]
    └── compute.os (local: ./os)
        └── [2 resources]

✨ Graft manifest saved to scaffold.graft.hcl
```

Simple, but essential. Now users can see the hierarchy at a glance and start writing overrides immediately—without ever opening the module source.

## Try It

```bash
go install github.com/ms-henglu/graft@latest
```

The workflow:

```bash
terraform init
graft scaffold    # See the module tree, generate starter manifest
# Edit manifest.graft.hcl
graft build       # Vendor, patch, and link
terraform plan    # Your patches are applied
```

Your `main.tf` never changes. When the upstream module releases a new version, bump the version, run `terraform init && graft build`, and your patches are reapplied.

No forks. No merge conflicts.

Check the [examples](https://github.com/ms-henglu/graft/tree/main/examples) for patterns like overriding values, injecting resources, removing attributes, and adding lifecycle rules.

---

The full code is at [github.com/ms-henglu/graft](https://github.com/ms-henglu/graft).

If you try it and hit issues—or have ideas—[open an issue](https://github.com/ms-henglu/graft/issues). I'd love to hear what breaks.

Happy patching. 🌱


