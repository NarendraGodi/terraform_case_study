----------

## 🧠 Why labels might cause VM recreation in modules:

### ✅ Normally:

-   `labels` are a mutable field on `google_compute_instance`.
    
-   Terraform should update them in-place with a **PATCH request** to GCP.
    

----------

### ❌ But in Modules:

When used in a module, a change like this:

```diff
labels = {
  "env" = "staging"  # changed from "dev"
}

```

can sometimes cause **recreation**, due to:

----------

### 🔸 1. **Use of `for_each` or computed keys**

If the `google_compute_instance` inside your module is using `for_each`, and the key for each instance depends (even indirectly) on labels or something that changed, Terraform treats it as a **new resource**.

```hcl
for_each = {
  for name, config in var.instances : name => config
}

```

Even slight changes in the shape of `for_each` inputs = recreation.

----------

### 🔸 2. **Labels passed as dynamic or computed values**

If `labels` are passed from another module or depend on outputs that change often (e.g., using `merge`, `locals`, or computed expressions), even a no-op plan can lead Terraform to believe the object has changed.

Terraform is very strict about comparing map structures — even a **change in key order** or a value being marked as computed can result in:

```
-/+ google_compute_instance.vm (new resource required)

```

----------

### 🔸 3. **Custom diff logic or provider bugs**

Terraform's GCP provider has had historical bugs around label diffs, especially when the map structure is not clean or uses computed values. This can cause false-positive recreations.

----------

## ✅ How to prevent VM recreation due to labels

### ✔️ Use `lifecycle.ignore_changes`

Inside your module, modify the resource like:

```hcl
resource "google_compute_instance" "vm" {
  ...

  lifecycle {
    ignore_changes = [labels]
  }
}

```

This prevents label changes from triggering any action — **but also means they won’t be updated at all**, so use with care.

----------

### ✔️ Ensure clean, consistent label values

Avoid passing label maps like:

```hcl
labels = merge(local.base_labels, var.env_labels)

```

If you must merge, use `terraform console` or output the result to confirm no computed values remain.

----------

### ✔️ Avoid dependency loops in modules

Try to avoid passing labels or metadata from modules whose outputs change frequently. Stick to plain maps when possible.

----------

## 🔚 Summary

-   Labels are **technically mutable**, but in modules they may still cause recreation due to:
    
    -   `for_each` usage
        
    -   dynamic or computed values
        
    -   subtle diff issues in Terraform
        
-   You can use `ignore_changes`, or restructure inputs to avoid computed maps.
    
