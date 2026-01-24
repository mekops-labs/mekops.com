# MekOps.com

> **The Digital Headquarters of MekOps.**
>
> *Where Cloud-Native meets Bare Metal.*

![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)
![Built With: Hugo](https://img.shields.io/badge/Built%20With-Hugo-0079d3.svg)
![Theme: Clarity](https://img.shields.io/badge/Theme-Clarity-success.svg)

## 📡 About The Project

This repository contains the source code for [**mekops.com**](https://mekops.com), my personal portfolio and engineering blog.

## 🛠 Tech Stack

* **Generator:** [Hugo](https://gohugo.io) (Fast, static, secure).
* **Theme:** [Hugo Clarity](https://github.com/chipzoller/hugo-clarity) (Enterprise-grade, documentation-focused design).
* **Comments:** [Giscus](https://giscus.app) (GitHub Discussions integration).
* **Analytics:** [GoatCounter / Cloudflare] (Privacy-first).

## 🚀 Quick Start

To run this site locally:

1.  **Clone the repository:**
    ```bash
    git clone --recurse-submodules [https://github.com/mekops-labs/mekops.com.git](https://github.com/mekops-labs/mekops.com.git)
    cd mekops.com
    ```

2.  **Update Theme (Optional):**
    If the theme submodule is empty:
    ```bash
    git submodule update --init --recursive
    ```

3.  **Run Hugo Server:**
    ```bash
    hugo server -D
    ```

4.  **View:** Open `http://localhost:1313` in your browser.

## 📂 Customization & Overrides

* `config/_default`: Main site configuration (Menus, Params).
* `static/`: Images, logos, and global assets.
* `layouts/`: Theme overrides (e.g., custom Giscus partials).

## © License

The content of this website (blog posts, images, documentation) is licensed under the **Creative Commons Attribution 4.0 International License (CC BY 4.0)**.

The Clarity theme is licensed under **MIT**.
