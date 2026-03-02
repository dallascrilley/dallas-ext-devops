---
description: Deploy documentation or static site to GitHub Pages
allowed-tools: Bash(git:*), Bash(gh:*)
---

# Deploy Docs

Deploy documentation: $ARGUMENTS

## Instructions

1. **Identify build command**:
   - Check for build scripts in `package.json`, `Makefile`, or framework config
   - Common: `npm run build`, `make docs`, `mkdocs build`, `hugo`

2. **Build verification**:
   ```bash
   # Run the build
   <build command>

   # Verify output directory exists and has content
   ls -la <output-dir>/
   ```

3. **Pre-deploy checks**:
   - [ ] Build succeeded without errors
   - [ ] Output directory is not empty
   - [ ] No broken links (if link checker available)
   - [ ] Version/date is current

4. **Deploy**:
   ```bash
   # GitHub Pages via gh-pages branch
   gh api repos/{owner}/{repo}/pages -X POST -f source='{"branch":"gh-pages","path":"/"}' 2>/dev/null || true

   # Push build output to gh-pages branch
   git subtree push --prefix <output-dir> origin gh-pages
   ```

   Or if using GitHub Actions:
   ```bash
   # Trigger the deploy workflow
   gh workflow run deploy.yml
   ```

5. **Verify deployment**:
   - Check GitHub Pages URL is responding
   - Report the URL to the user
