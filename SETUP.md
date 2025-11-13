# MoneyFlow Website Setup

This repository contains the public website for the MoneyFlow project.

## Website URL

Once deployed, the website will be available at:
**https://jinliangwii.github.io/MoneyFlowWebPage/**

## Initial Setup

### 1. Create GitHub Repository

1. Go to GitHub and create a new repository named `MoneyFlowWebPage`
2. **Do NOT** initialize it with a README, .gitignore, or license (we already have these)

### 2. Push to GitHub

```bash
cd /Users/jinliangwei/Developer/MoneyFlowWebPage

# Add the remote repository (replace with your actual GitHub username if different)
git remote add origin git@github.com:jinliangwii/MoneyFlowWebPage.git

# Push to GitHub
git branch -M main
git push -u origin main
```

### 3. Enable GitHub Pages

1. Go to your repository on GitHub: https://github.com/jinliangwii/MoneyFlowWebPage
2. Navigate to **Settings** → **Pages** (in the left sidebar)
3. Under **Build and deployment** section:
   - Set **Source** to: `GitHub Actions`
   - Click **Save**
4. **Also check Workflow Permissions:**
   - Go to **Settings** → **Actions** → **General**
   - Under **Workflow permissions**, select: **Read and write permissions**
   - Check **Allow GitHub Actions to create and approve pull requests**
   - Click **Save**

### 4. Wait for Deployment

After enabling Pages, the GitHub Actions workflow will automatically run and deploy your site. You can monitor the progress in the **Actions** tab.

## Local Development

```bash
# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve --baseurl /MoneyFlowWebPage

# Visit http://localhost:4000/MoneyFlowWebPage/
```

## Updating Documentation

To update the documentation:

1. Edit files in the `Docs/` folder
2. Commit and push changes:
   ```bash
   git add Docs/
   git commit -m "Update documentation"
   git push origin main
   ```
3. The website will automatically rebuild and deploy

## Structure

- `index.md` - Homepage
- `docs.md` - Documentation index page
- `Docs/` - All documentation files (copied from main MoneyFlow repository)
- `_config.yml` - Jekyll configuration
- `.github/workflows/pages.yml` - GitHub Actions deployment workflow
- `Gemfile` - Ruby dependencies

## Related Repository

- Main project: [MoneyFlow](https://github.com/jinliangwii/MoneyFlow)

