# Simple Node.js CI/CD Demo

A minimal Express app used purely as a vehicle to show a complete pipeline:

```
Git commit -> GitHub -> Jenkins -> Docker build -> DockerHub push -> Kubernetes deploy
```

Deliberately kept small — no database, no auth, no multi-stage Dockerfile —
so nothing distracts from watching the *pipeline* run.

Uses Docker Desktop's built-in Kubernetes (no Minikube/kind) — just Docker
and Kubernetes, since Docker Desktop bundles both.

## Prerequisites (all on your trainer laptop)

| Tool | Notes |
|---|---|
| Node.js 20+ | to run the app locally if you want to sanity-check it first |
| Docker Desktop | with Kubernetes enabled (Settings → Kubernetes → Enable Kubernetes) |
| kubectl | pointed at the `docker-desktop` context (`kubectl config use-context docker-desktop`) |
| Jenkins | native (WAR), **not** in Docker — avoids Docker-in-Docker/socket-mount issues |
| DockerHub account | free tier is fine |
| GitHub repo | push this project to your own repo first |

## One-time setup (do this before class, not live)

1. **Push this project to GitHub**
   ```bash
   cd ibm-cicd-demo
   git init && git add . && git commit -m "initial commit"
   git branch -M main
   git remote add origin https://github.com/YOUR_GITHUB_USERNAME/ibm-cicd-demo.git
   git push -u origin main
   ```

2. **Create a DockerHub access token** (DockerHub > Account Settings > Security)

3. **Add DockerHub credentials in Jenkins**
   Manage Jenkins → Credentials → System → Global → Add Credentials
   - Kind: Username with password
   - Username: your DockerHub username
   - Password: the access token
   - ID: `dockerhub-creds` (must match the Jenkinsfile)

4. **Edit placeholders**
   - `Jenkinsfile`: `IMAGE_NAME`, GitHub URL
   - `k8s/deployment.yaml`: image name

5. **Verify kubectl is pointed at Docker Desktop's cluster**
   ```bash
   kubectl config current-context   # should print: docker-desktop
   kubectl get nodes                # should show one node, Ready
   ```

6. **Pre-create the Deployment/Service once, manually** (the pipeline only
   updates the image after this — it doesn't create the Deployment from scratch)
   ```bash
   kubectl apply -f k8s/
   ```

7. **Create the Jenkins Pipeline job**
   New Item → Pipeline → Pipeline script from SCM → Git → your repo URL,
   script path `Jenkinsfile`. Enable "Poll SCM" (e.g. `H/2 * * * *`) if you
   want auto-trigger on push, since there's no public webhook on a laptop.

8. **Sanity-run the pipeline once before class** so you're not debugging live.

## Running it locally (optional, outside the pipeline)

```bash
cd app
npm install
npm test
node server.js   # http://localhost:3000
```

See `demo-script.md` for the live walkthrough.
