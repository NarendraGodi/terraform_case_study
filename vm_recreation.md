
---
🧑‍💻, 💻, 🔧: Code referenced from https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/tree/master/modules/compute-vm
## ✅ Involving Factors:

Problem statement1:

- A VM module references `module.iam-service-account.email`.
- A new **IAM binding (group added to service account users)** caused **VM recreation**.
- `depends_on` is also defined on the VM module.

---

## 🔍 Likely Root Cause:

Even though the **email** of the service account didn't change, **Terraform saw a change to the `iam-service-account` module**, and **re-evaluated** its outputs (like `email`). This **can trigger recreation** of resources that **consume that output**, like your VM.

This happens because Terraform **doesn’t track attribute-level changes very well inside modules** — when a module's internal state changes (e.g., IAM bindings), it might consider **the entire module output as updated**, even if the value hasn't changed.

And if `module.iam-service-account.email` is passed directly into the VM definition, the VM resource might be **marked for recreation**.

---

## 🔁 What reinforces this?

You also mentioned using `depends_on` in the VM module, which adds **explicit dependency**, but that *alone* doesn’t cause recreation. However:

> ⚠️ If Terraform detects any *change* in the **input values** to the `vm-managed-sa-example2` module (like `service_account.email`), and the VM resource inside that module treats it as immutable, **the VMs will be recreated**.

---

## ✅ Solution Options:

### **1. Isolate IAM bindings from the Service Account resource**

Use a separate module or resource to manage IAM bindings so changes to them don't affect the service account resource itself (and its outputs).

```hcl
# service-account.tf
resource "google_service_account" "default" {
  account_id = "my-sa"
  ...
}

output "email" {
  value = google_service_account.default.email
}

# iam-bindings.tf (in a separate module/file)
resource "google_service_account_iam_binding" "user" {
  service_account_id = google_service_account.default.name
  role               = "roles/iam.serviceAccountUser"
  members            = var.members
}
```

This way, **changes to IAM bindings don't touch the service account resource**, avoiding re-evaluation of its outputs.

---

### **2. Use `terraform plan` to verify output value diffs**

In your case, try running:

```bash
terraform plan -detailed-exitcode
```

And examine if the plan shows any change to the value of `module.iam-service-account.email`. Even if the string value doesn’t change, **Terraform may detect a diff** due to how modules are evaluated.

---

### **3. Cache or hardcode stable values when safe**

If you’re confident the email of the SA won’t change, you could consider using a `local` or hardcoding it in the VM module to break the dynamic dependency.

```hcl
locals {
  sa_email = module.iam-service-account.email
}

module "vm" {
  ...
  service_account = {
    email = local.sa_email
  }
}
```

---

### Summary:

- VM was recreated because **Terraform re-evaluated the module output (`email`)** when IAM bindings changed.
- That output was passed into the VM module, which triggered **resource recreation**.
- To fix this, **separate IAM bindings**, avoid unnecessary dependencies in outputs, and/or use stable values when possible.

This is a very sharp observation — and the behavior you're seeing often surprises even experienced Terraform users. Let's unpack the scenario carefully.

----------

## 🧠 Data Getting destroyed:

-   You're using a custom module that:
    
    -   Creates a blank persistent disk.
        
    -   Attaches it to a VM using a `google_compute_attached_disk` resource (external to the `google_compute_instance`).
        
-   Inside the module, you have:
    

```hcl
lifecycle {
  ignore_changes = [attached_disk]
}

```

-   But despite this, **the attached disk was recreated** when the service account changed and the VM was recreated.
    

----------

## 🔍 Why the `lifecycle` rule didn’t stop disk recreation

### Key Concept:

> **The lifecycle rule only applies to the resource it's defined on.**

So if you're seeing:

```hcl
resource "google_compute_instance" "vm" {
  ...
  lifecycle {
    ignore_changes = [attached_disk]
  }
}

```

It tells Terraform:

-   “If the `attached_disk` block in _this resource_ changes, don’t trigger recreation.”
    

But your setup uses **`google_compute_attached_disk`**, a **separate resource**, not an `attached_disk` block inside the instance. And **that resource doesn’t inherit the VM’s lifecycle rules**.

----------

### So what happens?

1.  Your VM gets recreated due to service account reference change.
    
2.  Terraform destroys the old VM.
    
3.  Since the **disk is attached via a `google_compute_attached_disk` resource**, Terraform must:
    
    -   **Detach** the disk from the destroyed VM.
        
    -   **Reattach** it to the new VM (with a new `instance_id`).
        
4.  That reattachment = **destroy and recreate** of `google_compute_attached_disk`.
    

Even if `ignore_changes` were used on the disk attachment resource (e.g., to ignore `instance`), it wouldn’t help when the **instance itself changes** — because that change causes **resource destruction**, not just an update.

----------

## ✅ What You Can Do

### Option 1: Use `google_compute_instance.attached_disk` instead of a separate resource

If feasible, you could define the disk attachment inline:

```hcl
resource "google_compute_instance" "vm" {
  name = "example"
  ...

  attached_disk {
    source = google_compute_disk.data_disk.id
    mode   = "READ_WRITE"
  }

  lifecycle {
    ignore_changes = [attached_disk]
  }
}

```

Then the `ignore_changes` will correctly suppress unnecessary updates.

----------

### Option 2: Split disk attachment from instance lifecycle (advanced)

If you must use `google_compute_attached_disk`, and want to **avoid destroying and re-creating** it, consider:

-   **Using a `null_resource` and local-exec** to attach the disk _imperatively_, which avoids tracking in Terraform (but is harder to manage).
    
-   **Avoiding dynamic dependencies** (like service account output) to reduce instance recreation in the first place.
    

----------

### Option 3: Acceptable tradeoff

If the disk is not wiped (e.g., it’s a persistent blank disk), and the `google_compute_attached_disk` simply reattaches the same disk to a new VM, this might be acceptable in practice — **data is not lost**, and the infra behaves as expected.

----------

## 🔚 TL;DR

-   `lifecycle { ignore_changes = [attached_disk] }` only affects inline blocks in `google_compute_instance`, **not** separate `google_compute_attached_disk` resources.
    
-   When the VM is recreated (e.g., due to SA changes), any attached disk must be reattached — which is why you see `google_compute_attached_disk` being recreated.
    
-   To prevent that, consider:
    
    -   Inline disk attachment + lifecycle rules
        
    -   Reducing dependencies that trigger VM recreation
        
    -   Accepting the reattachment if data is preserved
        
