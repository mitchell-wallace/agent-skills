---
name: vercel-preview
description: Deploy web applications to Vercel preview environments autonomously using the Vercel CLI
---

# Vercel Preview Deployment Skill

Deploy web applications to Vercel preview environments autonomously.

## Prerequisites

- `VERCEL_TOKEN` environment variable must be set (obtained from Vercel dashboard → Account Settings → Tokens)
- Network access enabled for `vercel.com` and `api.vercel.com`
- Node.js available in the environment

## Security Rules

**CRITICAL: Never attempt to read, print, echo, or inspect the contents of VERCEL_TOKEN.**

The only permitted verification is this exact command:

```bash
[ -z "${VERCEL_TOKEN:-}" ] && echo "VERCEL_TOKEN_STATUS=missing" || echo "VERCEL_TOKEN_STATUS=set"
```

If the token is missing or invalid (determined by Vercel API errors), inform the user and stop. Do not attempt to:
- Ask for the token
- Search for the token
- Generate a new token
- Debug token-related authentication issues

## Deployment Process

### Step 1: Verify Token Exists

Run the exact command above. If status is `missing`, respond:

> "The VERCEL_TOKEN environment variable is not set. Please add it to your Claude Code environment settings. You can generate a token at https://vercel.com/account/tokens"

Then stop.

### Step 2: Install Vercel CLI

```bash
npm install -g vercel
```

### Step 3: Gather Git Context

Capture branch and commit info for Vercel's dashboard:

```bash
BRANCH=$(git branch --show-current 2>/dev/null || echo "HEAD")
COMMIT=$(git rev-parse HEAD 2>/dev/null || echo "unknown")
```

If git is not initialized or these fail, proceed without metadata.

### Step 4: Deploy with Branch Metadata

From the project root directory:

```bash
vercel deploy --yes --token="$VERCEL_TOKEN" \
  --meta gitBranch="$BRANCH" \
  --meta gitCommit="$COMMIT" 2>&1
```

**Flags explained:**
- `--yes`: Skip all interactive prompts, use defaults
- `--token`: Authenticate non-interactively
- `--meta`: Associate deployment with git branch/commit in Vercel dashboard

### Step 5: Handle Build Failures (Adaptive Configuration)

If the deployment fails due to build errors (not auth errors), follow this process:

1. **Analyze the error output** to determine the cause (wrong build command, wrong output directory, missing dependencies, etc.)

2. **Inspect the repository** to understand the project structure—check `package.json` scripts, existing config files, and source layout

3. **Check if `vercel.json` exists** in the project root

4. **Create or update `vercel.json`** with appropriate settings based on your analysis:

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "installCommand": "npm install",
  "framework": null
}
```

5. **Retry the deployment** after creating/updating config

6. **Repeat up to 5 times** if builds continue to fail—adjust configuration based on each error

7. **After 5 failed attempts**, share the errors with the user and ask for guidance

Example adaptive flow:
```bash
# First attempt fails with "Build failed: command not found: vite"
# Create vercel.json:
cat > vercel.json << 'EOF'
{
  "buildCommand": "npx vite build",
  "outputDirectory": "dist"
}
EOF

# Retry deployment
vercel deploy --yes --token="$VERCEL_TOKEN" \
  --meta gitBranch="$BRANCH" \
  --meta gitCommit="$COMMIT" 2>&1
```

### Step 6: Capture and Share Deployment URLs

The deploy command outputs both an Inspect URL and a Preview URL. **Always share both with the user.**

Example success output:
```
Vercel CLI 37.x.x
🔍  Inspect: https://vercel.com/team/project/deployment-id
✅  Preview: https://project-abc123.vercel.app
```

Share with the user:
- **Inspect URL**: Links to the Vercel dashboard for this deployment (build logs, settings, errors)
- **Preview URL**: The live preview of the deployed application

The Inspect link is especially useful if something looks wrong—the user can check build logs and deployment details there.

### Step 7: Report Configuration Changes

If you created or modified `vercel.json`, inform the user:

> "I created/updated `vercel.json` with the following configuration to fix the build:
> ```json
> { ... }
> ```
> You may want to commit this file to your repository."

## Handling Specific Scenarios

### First-time project deployment

When deploying a project that hasn't been linked to Vercel before, the CLI with `--yes` will:
1. Create a new Vercel project using the directory name
2. Auto-detect the framework
3. Use default build settings

This is expected behavior. The project will appear in the user's Vercel dashboard.

### Project already exists on Vercel

If a `.vercel/project.json` file exists in the repo, the CLI will use those settings automatically.

### Specifying team/scope

If deploying to a team account rather than personal:

```bash
vercel deploy --yes --scope=team-slug --token="$VERCEL_TOKEN" 2>&1
```

### Authentication errors

If you see errors like:
- `Error: Invalid token`
- `Error: The provided token is not valid`
- `401 Unauthorized`

Respond:

> "The Vercel deployment failed due to an authentication error. Please verify that your VERCEL_TOKEN is valid and has not expired. You can generate a new token at https://vercel.com/account/tokens"

Then stop. Do not attempt further debugging.

### Build failures

Build failures are handled by the adaptive configuration process in Step 5. The skill will:
1. Analyze the error
2. Inspect the repo to understand the project
3. Create/update `vercel.json` with appropriate settings
4. Retry up to 5 times, adjusting config based on each error
5. After 5 failures, report to user and ask for guidance

### Linking to existing project

If the user wants to deploy to a specific existing Vercel project:

```bash
vercel link --yes --project=project-name --token="$VERCEL_TOKEN"
vercel deploy --yes --token="$VERCEL_TOKEN" 2>&1
```

## Production Deployments

This skill is intended for **preview deployments only**. If a user explicitly requests a production deployment, use:

```bash
vercel deploy --prod --yes --token="$VERCEL_TOKEN" 2>&1
```

But confirm with the user first before deploying to production.

## Environment Variables for the Deployed App

If the app requires environment variables at build/runtime, they can be passed inline:

```bash
vercel deploy --yes --token="$VERCEL_TOKEN" \
  --build-env KEY=value \
  --env KEY=value 2>&1
```

Or instruct the user to configure them in the Vercel dashboard for persistent configuration.

## Quick Reference

| Action | Command |
|--------|---------|
| Check token | `[ -z "${VERCEL_TOKEN:-}" ] && echo "missing" \|\| echo "set"` |
| Get branch | `git branch --show-current` |
| Get commit | `git rev-parse HEAD` |
| Preview deploy | `vercel deploy --yes --token="$VERCEL_TOKEN" --meta gitBranch="$BRANCH"` |
| Production deploy | `vercel deploy --prod --yes --token="$VERCEL_TOKEN"` |
| Link to project | `vercel link --yes --project=name --token="$VERCEL_TOKEN"` |
| Deploy to team | `vercel deploy --yes --scope=team --token="$VERCEL_TOKEN"` |

## vercel.json Template

For projects needing custom build configuration:

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "installCommand": "npm install",
  "framework": null,
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

The `rewrites` rule is useful for SPAs that handle routing client-side.
