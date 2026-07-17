## Introduction

Suphle is designed around **modules, strict structure, and test-first validation**. This guide gets you from zero to your first working endpoint in minutes.

---

## Requirements

* **PHP 8.1+**
* **Composer**

---

## Installation

Launch your terminal into your system's web folder to install Suphle's starter bootstrapper:

```bash
cd "C:\wamp64\www"

composer create-project nmeri/suphle-starter AwesomeProject

cd AwesomeProject

```

---

## Create Your First Module

Suphle applications are built as **modular units**, not a flat folder of unstructured controllers. Your module can be named anything, but for this guide, let's create a `Products` module:

```bash
php suphle modules:create Products --module_descriptor="\AllModules\Products\Meta\ProductsDescriptor"

```

This scaffolds a fully isolated feature unit right out of the box containing:

* **Routing** (Coordinator)
* **Tests**
* **Configuration**
* **Module descriptor**

Before exploring these files, let's connect this fresh module to the rest of the application so it can begin intercepting requests.

---

## Register the Module

Make the following adjustments to the `AllModules\PublishedModules.php` class in your project to register the new descriptor:

```diff
<?php

namespace AllModules;

use Suphle\Modules\ModuleHandlerIdentifier;
use Suphle\Hydration\Container;
+ use AllModules\Products\Meta\ProductsDescriptor;

class PublishedModules extends ModuleHandlerIdentifier {
    
    public function getModules ():array {
+       return [new ProductsDescriptor(new Container)];
    }
}
?>

```

---

## Verifying Project Initialization

There is one default URL predefined in the template. To access it, return to your terminal and spin up the application server:

```bash
php suphle server:start AllModules

```

Open your browser and visit the default URL: [http://localhost:8080/products/hello](http://localhost:8080/products/hello). You should see the following JSON payload:

```json
{"message":"Hello World!"}

```

Congratulations? Well, not quite yet.

---

## Verify the Suphle Way (Tests First)

Checking a browser works, but it isn't repeatable or automated. In Suphle, we verify through test-first validation.

First, stop the running server in your terminal by pressing **Ctrl + C** (on Windows/Linux). Then, execute the automated test that accompanied your new installation:

```bash
./vendor/bin/phpunit AllModules/Products/Tests/ConfirmInstall.php

```

If everything is set up correctly, you will see a successful test output look exactly like this:

```text
PHPUnit 9.6.6 by Sebastian Bergmann and contributors.

.                                                               1 / 1 (100%)


Time: 00:01.077, Memory: 16.00 MB

OK (1 test, 2 assertions)

```

**Voila!** You now have a reproducible, irrefutable confirmation that your very first endpoint passes with flying colors.

---

## What Just Happened?

You didn’t just create a route—you created a **self-contained feature module**.

> ### 💡 Why This Matters
> 
> 
> In traditional flat-folder setups, making a major change feels like playing Jenga—you can never tell at a glance what might break elsewhere. Suphle bakes the **modular monolith** approach directly into the framework. By enforcing physical boundaries and explicit dependency management between modules, you can instantly see the exact impact of your changes without guessing.

Each module in Suphle:

* **Owns its own routes and logic** completely.
* **Is independently testable** out of the box.
* **Maintains strict architectural boundaries** so your codebase scales safely.

---

## Next Steps

Now that your application is running, feel free to dive into any chapter on the menu that catches your eye. Don't be satisfied with just knowing how to intercept an incoming payload—finishing the upcoming chapters will teach you how to master:

* How **Coordinators** handle incoming requests
* Utilizing **[PayloadStorage](/docs/v2/service-coordinators#Retrieving-request-input)** for clean input handling
* Offloading business logic into isolated **Services**
* Shaping your output with **Renderers**, alongside managing events and exceptions

Have a jolly ride!

---