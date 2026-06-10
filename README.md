# 🛍️ EasyShop — Modern E-commerce Platform

<div align="center">

[![Next.js](https://img.shields.io/badge/Next.js-14.1.0-black?style=for-the-badge&logo=next.js)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0.0-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![MongoDB](https://img.shields.io/badge/MongoDB-8.1.1-47A248?style=for-the-badge&logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![Redux](https://img.shields.io/badge/Redux-2.2.1-764ABC?style=for-the-badge&logo=redux&logoColor=white)](https://redux.js.org/)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-deployed-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

A production-grade, full-stack e-commerce platform with a complete CI/CD pipeline on AWS EKS.

![EasyShop Website Screenshot](./public/Deployed.png)

</div>

---

## ✨ Features

| Feature | Details |
|---|---|
| 🎨 Responsive UI | Dark/Light mode, mobile-first with Tailwind CSS |
| 🔐 Auth | Secure JWT + NextAuth session management |
| 🛒 Cart | Real-time cart state via Redux Toolkit |
| 🔍 Search | Advanced product search and category filtering |
| 💳 Checkout | Multi-step checkout flow |
| 👤 Profiles | User accounts with order history |
| 📦 Categories | Electronics, Grocery, Clothing, Furniture, Beauty & more |

---

## 🏗️ Architecture

EasyShop uses a classic three-tier architecture, fully containerised and deployed on Kubernetes.

```
┌─────────────────────────────────────────────┐
│            Presentation Tier                │
│  Next.js Components · Redux · Tailwind CSS  │
└───────────────────┬─────────────────────────┘
                    │ HTTP
┌───────────────────▼─────────────────────────┐
│             Application Tier                │
│  Next.js API Routes · Auth · Business Logic │
└───────────────────┬─────────────────────────┘
                    │ Mongoose ODM
┌───────────────────▼─────────────────────────┐
│               Data Tier                     │
│          MongoDB · CRUD · Validation        │
└─────────────────────────────────────────────┘
```

---

## 🚀 Tech Stack

### Application
[![Next.js](https://img.shields.io/badge/Next.js-000000?style=flat-square&logo=next.js)](https://nextjs.org/)
[![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=flat-square&logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![Redux](https://img.shields.io/badge/Redux-764ABC?style=flat-square&logo=redux&logoColor=white)](https://redux.js.org/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)](https://tailwindcss.com/)

### Infrastructure & CI/CD
[![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat-square&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)](https://www.jenkins.io/)
[![ArgoCD](https://img.shields.io/badge/Argo_CD-EF7B4D?style=flat-square&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Trivy](https://img.shields.io/badge/Trivy-1904DA?style=flat-square&logo=aquasecurity&logoColor=white)](https://aquasecurity.github.io/trivy/)

---

## 📦 Project Structure

```
easyshop/
├── src/
│   ├── app/              # Next.js App Router pages & layouts
│   ├── components/       # Reusable React components
│   └── lib/
│       ├── auth/         # NextAuth + JWT logic
│       ├── db/           # Mongoose connection
│       └── features/     # Redux slices
├── kubernetes/           # K8s manifests (namespace → ingress)
├── terraform/            # IaC for VPC, EKS, EC2 (Jenkins)
├── scripts/              # DB migration (ts-node + Docker)
├── Dockerfile            # Multi-stage production build
├── docker-compose.yml    # Local dev stack
└── Jenkinsfile           # CI pipeline (Shared Library)
```

---

## ⚡ Quick Start (Local — Docker Compose)

```bash
# 1. Clone
git clone https://github.com/rohandeb2/Easy-shop-E-commerce.git
cd Easy-shop-E-commerce

# 2. Create .env.local
cp .env .env.local
# Edit NEXTAUTH_SECRET and JWT_SECRET:
openssl rand -base64 32   # → NEXTAUTH_SECRET
openssl rand -hex 32      # → JWT_SECRET

# 3. Run
docker compose up -d

# 4. Open
open http://localhost:3000
```

> See [DEPLOYMENT.md](DEPLOYMENT.md) for the full AWS EKS production deployment guide.

---

## 🔧 Troubleshooting

**MongoDB not reachable**
```bash
docker ps
docker logs easyshop-mongodb
docker network inspect easyshop-network
```

**Build errors / strange behaviour**
```bash
rm -rf .next
npm install
docker compose down -v && docker compose up -d
```

---

## 🤝 Contributing

1. Fork the repository
2. Create a branch: `git checkout -b feature/my-feature`
3. Commit: `git commit -m 'feat: add my feature'`
4. Push: `git push origin feature/my-feature`
5. Open a Pull Request

---

<div align="center">
  Made with ❤️ by <a href="https://github.com/rohandeb2"><b>Ruhon Deb</b></a>
  &nbsp;·&nbsp;
  <a href="DEPLOYMENT.md">📖 Full Deployment Guide</a>
</div>
