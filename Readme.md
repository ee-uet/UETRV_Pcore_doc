# UETRV_Pcore Documentation

[![Documentation Status](https://readthedocs.org/projects/uetrv-pcore-doc/badge/?version=main)](https://uetrv-pcore-doc.readthedocs.io/en/main/?badge=main)

This repository contains the documentation for the **UETRV_Pcore** project, generated using [Sphinx](https://www.sphinx-doc.org/) with [Markdown](https://www.markdownguide.org/) support via [MyST Parser](https://myst-parser.readthedocs.io/).

📘 **Read the full documentation:**  
👉 [https://uetrv-pcore-doc.readthedocs.io/en/main/](https://uetrv-pcore-doc.readthedocs.io/en/main/)

---

## 🛠️ Build Docs Locally

To build the documentation locally:

1. Clone the repository:

   ```bash
   git clone https://github.com/ee-uet/UETRV_Pcore_doc.git
   cd UETRV_Pcore_doc
   ```

2. Create a virtual environment and install dependencies:

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r docs/requirements.txt
   ```

3. Build the HTML documentation:

   ```bash
   cd docs
   make html
   ```

4. Open `_build/html/index.html` in your browser.

---

## 🧱 Tech Stack

* **Sphinx**
* **MyST Parser** for Markdown support
* **nbsphinx** for Jupyter notebooks
* **sphinx\_rtd\_theme** (Read the Docs theme)

---

## 📂 Directory Structure

```
UETRV_Pcore_doc/
├── docs/
│   ├── source/
│   │   └── design_document/
│   │       └── index.md
│   │   └── images/
│   │   └── user_guide/
│   │       └── index.md
│   │   ├── index.md
│   ├── conf.py
│   ├── Makefile
├── .readthedocs.yml
└── README.md
└── requirements.txt
```

---

## 🧪 Continuous Documentation

Documentation is automatically built and deployed via [Read the Docs](https://readthedocs.org/). Any changes pushed to the `main` branch are reflected here:

🔗 [https://uetrv-pcore-doc.readthedocs.io/en/main/](https://uetrv-pcore-doc.readthedocs.io/en/main/)

---

## 👥 Contributors

* [@ee-uet](https://github.com/ee-uet) and the UET RISC-V Team

---

## 📜 License

This project is licensed under the MIT License.
