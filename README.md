# Cloud Networking and Resilience

Supporting repository for the Apress title  
**_Cloud Networking and Resilience_ (Apress, 2026)**  
by **Cristian Critelli**

---

## 📘 About This Repository
This repository contains supporting examples, configurations, and diagrams referenced throughout the book *Cloud Networking and Resilience*.  
It is organized to mirror the structure of the chapters, focusing on practical implementation of cloud networking and resilience patterns across AWS and hybrid environments.

---

### 🗂️ Structure
```
cloud-networking-resilience/
├── architectural-diagrams/   → Architecture diagrams (draw.io)
├── chapters/                 → Markdown notes and excerpts per chapter
├── code/                     → Configurations, JSON templates, scripts
├── docs/                     → Additional documentation or datasets
└── images/                   → Architecture diagrams (PNG)
```

---

## 🧩 Examples Included
- **DNS & Routing:** Route 53 failover JSONs, latency-based routing configs  
- **BGP Traffic Engineering:** Prepending, MED, and local-preference examples  
- **AWS Direct Connect & VPN:** Hybrid high-resilience reference models  
- **Resilience Automation:** Lambda failover scripts and ARC readiness checks  

---

## 🖼️ Images & Diagrams

- Each figure referenced in the book is stored in the `/images/` folder, each under their own chapters.  
- Each source file (.drawio) related to the figures in the book is stored in the `/architectural-diagrams/` folder, each under their own chapters.

<p align="center">
  <img src="images/chapter-6/Fig.6.14.png" alt="Figure 6.14 – Containment and Recovery in Active-Active Systems" width="700"/>
</p>

<p align="center">
  <em>Figure 6.14 – Containment and Recovery in Active-Active Systems</em><br/>
  <a href="https://github.com/crcritel/cloud-networking-resilience/blob/main/architectural-diagrams/chapter-6/Fig.6.14.containment%26recovery-a-a.drawio?raw=true" download="Fig.6.14.containment&recovery-a-a.drawio">
    <img src="https://img.shields.io/badge/💾%20Save%20this%20diagram-Draw.io%20source-blue?style=for-the-badge" alt="Download Draw.io source"/>
  </a><br/>
  <em>(If your browser opens the XML, right-click → “Save Link As…” and it will download as a .drawio file.)</em>
</p>

---

Copyright © 2025 Cristian Critelli. All rights reserved.  
This repository is intended as a companion for the book *Cloud Networking and Resilience* (Apress 2026).  
You may view and reference this material for educational and non-commercial purposes only.  
Redistribution or modification without permission from the author or Apress Media LLC is prohibited.
