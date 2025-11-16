# AXCL User Guide

[Web Preview](https://axcl-docs.readthedocs.io/zh-cn/latest/)

## 1. Project Background

**AXCL** is a Host API for PCIe EP products based on the AX650N.

- Provides APIs and examples for using AXCL with NPU computation;
- Encourages community-driven documentation maintenance.

## 2. Local build guide

### 2.1 git clone

```bash
git clone https://github.com/AXERA-TECH/axcl-docs.git
```

Directory structure:

```bash
.
├── LICENSE
├── Makefile
├── README.md
├── build
│   ├── doctrees
│   └── html
├── requirements.txt
├── res
│   ├── M2_YUNJI_DSC05130.jpg
│   ├── axcl_architecture.svg
│   ├── axcl_concept.svg
│   ├── centos_dmsg_grep_cma.png
│   ├── centos_grub_info.png
│   ├── centos_selinux.png
│   ├── imagenet_cat.jpg
│   ├── transcode_ppl.png
│   ├── voc_dog.jpg
│   ├── voc_dog_yolov5s_out.jpg
│   ├── voc_horse.jpg
│   └── voc_horse_yolov5s_out.jpg
└── source
    ├── axcl_error_lookup.html
    ├── conf.py
    ├── doc_guide_axcl_api.md
    ├── doc_guide_faq.md
    ├── doc_guide_hardware.md
    ├── doc_guide_quick_start.md
    ├── doc_guide_samples.md
    ├── doc_guide_setup.md
    ├── doc_introduction.md
    ├── doc_update_info.md
    ├── index.rst
    └── media
```

### 2.2 Build

Install dependencies

```bash
pip install -r requirements.txt
```

Run the following commands from the project root

```bash
$ make clean
$ make html
```

### 2.3 Local preview

After building, open `build/html/index.html` in your browser to preview the documentation

## 3. References

This project is based on Sphinx. For more information: https://www.sphinx-doc.org/en/master/

## 4. Online publishing

Hosted on ReadTheDocs using the ReadTheDocs platform.
