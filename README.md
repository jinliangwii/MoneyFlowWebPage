# MoneyFlow Website

This is the public website repository for the MoneyFlow project.

## Website URL

The website is available at: **https://jinliangwii.github.io/MoneyFlowWebPage/**

## Setup

This website is built with Jekyll and automatically deployed via GitHub Actions.

### Local Development

```bash
# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve --baseurl /MoneyFlowWebPage

# Visit http://localhost:4000/MoneyFlowWebPage/
```

## Structure

- `index.md` - Homepage
- `docs.md` - Documentation index
- `Docs/` - All documentation files
- `_config.yml` - Jekyll configuration
- `.github/workflows/pages.yml` - GitHub Actions deployment workflow

## Deployment

The website is automatically deployed when changes are pushed to the `main` branch via GitHub Actions.

## Related Repository

- Main project: [MoneyFlow](https://github.com/jinliangwii/MoneyFlow)
