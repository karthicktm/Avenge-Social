# Deployment Guide - Railway.com

This guide explains how to deploy the Avenge Social application to Railway.com.

## Prerequisites

1. A Railway.com account (sign up at https://railway.com)
2. Railway CLI (optional but recommended): `npm install -g @railway/cli`
3. Git repository with your code

## Quick Start

### Option 1: Deploy via Railway Dashboard (Recommended)

1. **Create a New Project**
   - Go to https://railway.com/new
   - Select "Deploy from GitHub repo"
   - Authorize Railway to access your GitHub account
   - Select your repository

2. **Add PostgreSQL Database**
   - In your Railway project dashboard, click "New"
   - Select "Database" → "PostgreSQL"
   - Railway will automatically create and configure the database
   - The `DATABASE_URL` environment variable will be automatically set

3. **Configure Environment Variables**
   - Click on your service (web app)
   - Go to the "Variables" tab
   - Add the following required variables:

   ```
   # Database (automatically set by Railway PostgreSQL)
   DATABASE_URL=postgresql://... (already configured)

   # Authentication & Security
   AUTH_SECRET=<generate with: openssl rand -base64 32>
   ENCRYPTION_SECRET=<generate with: openssl rand -base64 32>
   ADMIN_SECRET=<generate with: openssl rand -base64 32>

   # App URLs (will be provided after first deployment)
   NEXTAUTH_URL=https://your-app.railway.app
   NEXT_PUBLIC_APP_URL=https://your-app.railway.app

   # Optional: API Keys (or configure in app Settings)
   YOUTUBE_API_KEY=<your-youtube-api-key>
   OPENROUTER_API_KEY=<your-openrouter-api-key>
   RAPIDAPI_KEY=<your-rapidapi-key>
   ```

4. **Generate Secrets**
   Run these commands locally to generate secure secrets:
   ```bash
   openssl rand -base64 32  # AUTH_SECRET
   openssl rand -base64 32  # ENCRYPTION_SECRET
   openssl rand -base64 32  # ADMIN_SECRET
   ```

5. **Deploy**
   - Railway will automatically detect the Dockerfile and railway.json
   - The deployment will:
     1. Build the Docker image
     2. Run database migrations (`prisma migrate deploy`)
     3. Start the application
   - First deployment may take 3-5 minutes

6. **Update App URLs**
   - After the first deployment, Railway will provide a URL (e.g., `your-app.railway.app`)
   - Update these environment variables with your actual URL:
     - `NEXTAUTH_URL`
     - `NEXT_PUBLIC_APP_URL`
   - The app will automatically redeploy

### Option 2: Deploy via Railway CLI

1. **Install Railway CLI**
   ```bash
   npm install -g @railway/cli
   ```

2. **Login to Railway**
   ```bash
   railway login
   ```

3. **Initialize Project**
   ```bash
   railway init
   ```

4. **Add PostgreSQL**
   ```bash
   railway add --database postgresql
   ```

5. **Set Environment Variables**
   ```bash
   railway variables set AUTH_SECRET=$(openssl rand -base64 32)
   railway variables set ENCRYPTION_SECRET=$(openssl rand -base64 32)
   railway variables set ADMIN_SECRET=$(openssl rand -base64 32)
   ```

6. **Deploy**
   ```bash
   railway up
   ```

7. **Get Your URL**
   ```bash
   railway domain
   ```

8. **Update App URLs**
   ```bash
   railway variables set NEXTAUTH_URL=https://your-app.railway.app
   railway variables set NEXT_PUBLIC_APP_URL=https://your-app.railway.app
   ```

## Configuration Files

The application is already configured for Railway deployment:

### `railway.json`
Defines the build and deployment strategy:
- Uses Dockerfile for builds
- Runs `prisma migrate deploy` before each deployment
- Restarts on failure with max 10 retries

### `Dockerfile`
Multi-stage Docker build:
- Base: Node 22 Alpine
- Dependencies: Installs npm packages
- Builder: Generates Prisma client and builds Next.js
- Runner: Production-optimized image with health checks

### `next.config.ts`
- Standalone output mode for optimized Docker deployments
- Remote image patterns for YouTube, TikTok, Instagram

## Environment Variables Reference

| Variable | Required | Description | How to Generate |
|----------|----------|-------------|-----------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string | Auto-set by Railway |
| `AUTH_SECRET` | Yes | NextAuth session encryption | `openssl rand -base64 32` |
| `ENCRYPTION_SECRET` | Yes | API key encryption in DB | `openssl rand -base64 32` |
| `ADMIN_SECRET` | Yes | Admin API endpoint protection | `openssl rand -base64 32` |
| `NEXTAUTH_URL` | Yes | App base URL for auth redirects | Your Railway domain |
| `NEXT_PUBLIC_APP_URL` | Yes | Public app URL | Your Railway domain |
| `YOUTUBE_API_KEY` | No | YouTube Data API v3 key | Google Cloud Console |
| `OPENROUTER_API_KEY` | No | OpenRouter API key | OpenRouter dashboard |
| `RAPIDAPI_KEY` | No | RapidAPI key for transcripts | RapidAPI dashboard |

## Database Migrations

### Automatic Migrations (Recommended)
Railway runs `prisma migrate deploy` automatically before each deployment via the `preDeployCommand` in `railway.json`.

### Manual Migrations
If you need to run migrations manually:

```bash
# Using Railway CLI
railway run npx prisma migrate deploy

# Or connect to your Railway database
railway run npx prisma studio
```

### Creating New Migrations
1. Develop locally and create migrations:
   ```bash
   npx prisma migrate dev --name your_migration_name
   ```

2. Commit the migration files:
   ```bash
   git add prisma/migrations
   git commit -m "Add migration: your_migration_name"
   git push
   ```

3. Railway will automatically apply the migration on next deployment

## Monitoring and Logs

### View Logs
- **Dashboard**: Click on your service → "Deployments" → Click on a deployment
- **CLI**: `railway logs`

### Health Checks
The application includes a health check endpoint at `/api/health` that Railway uses to monitor the service.

### Metrics
Railway provides built-in metrics:
- CPU usage
- Memory usage
- Network traffic
- Response times

## Custom Domain (Optional)

1. Go to your service in Railway dashboard
2. Click "Settings" → "Domains"
3. Click "Custom Domain"
4. Enter your domain
5. Update your DNS with the provided CNAME record
6. Update environment variables:
   ```bash
   NEXTAUTH_URL=https://yourdomain.com
   NEXT_PUBLIC_APP_URL=https://yourdomain.com
   ```

## Troubleshooting

### Deployment Fails
- Check logs: `railway logs` or in the dashboard
- Verify all required environment variables are set
- Ensure DATABASE_URL is correctly configured

### Database Connection Issues
- Verify PostgreSQL service is running
- Check DATABASE_URL format
- Ensure migrations have run successfully

### Build Failures
- Check if all dependencies are in `package.json`
- Verify Dockerfile syntax
- Ensure Prisma schema is valid: `npx prisma validate`

### Application Crashes on Startup
- Check for missing environment variables
- Verify database migrations completed
- Review application logs for errors

## Costs

Railway pricing (as of 2024):
- **Hobby Plan**: $5/month base + usage
- **Pro Plan**: $20/month base + usage
- PostgreSQL: ~$5-10/month depending on usage
- See https://railway.com/pricing for latest pricing

## Security Best Practices

1. **Never commit secrets**: Use environment variables
2. **Rotate secrets regularly**: Update AUTH_SECRET, ENCRYPTION_SECRET periodically
3. **Use strong passwords**: For PostgreSQL and admin access
4. **Enable 2FA**: On your Railway account
5. **Review access logs**: Monitor for suspicious activity
6. **Keep dependencies updated**: Run `npm audit` regularly

## Rollback

If a deployment causes issues:

1. **Via Dashboard**:
   - Go to "Deployments"
   - Find a previous working deployment
   - Click "Redeploy"

2. **Via CLI**:
   ```bash
   railway rollback
   ```

## CI/CD

Railway automatically deploys when you push to your connected branch:

1. **Configure Auto-Deploy**:
   - Go to service settings
   - Set "Production Branch" (usually `main`)
   - Enable "Auto Deploy"

2. **Deploy on Push**:
   ```bash
   git push origin main
   ```
   Railway will automatically build and deploy

## Support

- Railway Docs: https://docs.railway.com
- Railway Discord: https://discord.gg/railway
- Application Issues: Create an issue in your repository
