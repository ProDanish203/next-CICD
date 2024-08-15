# CI/CD Pipeline

## Getting Started

### Create an SSH Key
```
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy -N ""
```
- Login your VPS using this key
- Create Secrets in your github repository for security
```
SSH_PRIVATE_KEY
SSH_HOST
```

- Create a new folder named as `.github/workflows`
- Create a file named as `ci.yml` for Continous Integration

```
name: Continous Integration

on: 
    push:
        branches:
            - main
    pull_request:
        branches:
            - main
jobs:
    build:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v2
              with: 
                fetch-depth: 0
            - name: Use Node.js
              uses: actions/setup-node@v2
              with:
                node-version: '22.2.0'

            - name: Install dependencies
              run: npm install

```
- This CI workflow automatically sets up the build environment, checks out the latest code, and installs dependencies whenever code is pushed to or a pull request is made against the main branch. This ensures that any changes are integrated smoothly and that the dependencies are up to date, helping to maintain the overall health and stability of the project.


### CD Pipeline
```
name: Build And Deploy To Digital Ocean
on:
  push:
    branches:
      - main
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Use Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '22.2.0'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy to Production
        if: github.ref == 'refs/heads/main'
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          SSH_HOST: ${{ secrets.SSH_HOST }}
        run: |
          mkdir -p ~/.ssh/
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts
          ssh -i ~/.ssh/id_rsa root@${{ secrets.SSH_HOST }} 'echo SSH connection successful'
          rsync -avz --delete ${{ github.workspace }}/.next ${{ github.workspace }}/package.json ${{ github.workspace }}/package-lock.json ${{ github.workspace }}/next.config.mjs root@${{ secrets.SSH_HOST }}:/root/next-CICD/
          ssh root@${{ secrets.SSH_HOST }} '
            export NVM_DIR="$HOME/.nvm" &&
            [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" &&
            cd /root/next-CICD &&
            npm ci --only=production &&
            pm2 restart site
          '
```
- This pipeline ensures that any changes pushed to the main branch are automatically built and deployed to a production server. It sets up the necessary environment, installs dependencies, builds the project, and then deploys the build artifacts to the server, where it updates the application and restarts it to reflect the new changes.