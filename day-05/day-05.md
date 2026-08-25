## 🚀 Day 05 — CI/CD: Build & Push Docker Images to GitHub Container Registry

Day 05 of my DevOps learning journey 🚀

Today I implemented a **GitHub Actions CI/CD workflow** to automatically build and push Docker images for my TrackForge application.

### 🔧 What I implemented

✅ GitHub Actions workflow
✅ Docker Buildx setup
✅ GitHub Container Registry (GHCR) authentication
✅ Backend Docker image build & push
✅ Frontend Docker image build & push
✅ SHA-based image tagging
✅ `latest` image tagging
✅ Automatic execution on every push to `main`

### 🐳 Images

The workflow publishes two Docker images:

* `trackforge-backend`
* `trackforge-frontend`

Each image gets two tags:

```text
latest
<github-commit-sha>
```

Using the commit SHA is especially useful because every deployed image can be traced back to an exact source-code version.

### 🔄 CI/CD Flow

```text
Developer Push
      ↓
GitHub Actions
      ↓
Checkout Code
      ↓
Docker Buildx
      ↓
Build Backend Image
      ↓
Build Frontend Image
      ↓
Push Images to GHCR
      ↓
Versioned Docker Images
```

### 📚 What I learned

This helped me understand how Docker images can be integrated into a CI/CD pipeline and how **GitHub Actions + GHCR** can automate container image delivery.

Next step: connecting these images with the Kubernetes deployment pipeline. ☸️

#DevOps #Docker #GitHubActions #CICD #Kubernetes #GHCR #CloudNative #DevOpsJourney #100DaysOfDevOps
