# CLAUDE.md

## Project Overview

This is a template for scheduling a Python script as a recurring cron job using GitHub Actions. The script runs on a configurable schedule, calls an external API, logs the response to `status.log`, and automatically commits and pushes the log file back to the repository.

## Repository Structure

```
python-github-action-template/
├── main.py                        # Main script executed by the GitHub Action
├── requirements.txt               # Python dependencies
├── .python-version                # Pinned Python version (3.11)
├── status.log                     # Rolling log file committed by the Action
├── README.md                      # User-facing setup guide
└── .github/
    └── workflows/
        └── actions.yml            # GitHub Actions workflow definition
```

## Key Files

### `main.py`
The entry point for the scheduled job. It:
- Sets up a rotating file logger writing to `status.log` (1 MB max, 1 backup)
- Reads `SOME_SECRET` from the environment (gracefully falls back if missing)
- On execution, fetches weather data from `https://weather.talkpython.fm/api/weather/` for Raleigh, NC
- Logs the temperature from the JSON response

Replace the weather API call with the actual business logic needed for a given use case.

### `.github/workflows/actions.yml`
Defines the GitHub Actions workflow:
- **Trigger**: Cron schedule — currently every Monday at 00:00 UTC (`0 0 * * 1`)
- **Steps**:
  1. Checkout repository
  2. Set up Python 3.11
  3. Install dependencies from `requirements.txt`
  4. Execute `main.py` with `SOME_SECRET` injected from GitHub Secrets
  5. Commit any changes (primarily `status.log`) with git identity `action@github.com`
  6. Push changes back to `main` using `ad-m/github-push-action@v0.6.0`

### `requirements.txt`
Currently pins `requests==2.28.1`. Add additional dependencies here as needed.

## Development Workflow

### Local Development
1. Install dependencies: `pip install -r requirements.txt`
2. Set environment variables for secrets: `export SOME_SECRET=your_value`
3. Run the script: `python main.py`
4. Check `status.log` for output

### Adding a New Secret
1. Go to the repository **Settings → Secrets and variables → Actions → New repository secret**
2. Add a new secret with the desired name (e.g., `MY_API_KEY`)
3. Reference it in `actions.yml` under the `env:` block of the execute step:
   ```yaml
   env:
     SOME_SECRET: ${{ secrets.SOME_SECRET }}
     MY_API_KEY: ${{ secrets.MY_API_KEY }}
   ```
4. Read it in `main.py`:
   ```python
   MY_API_KEY = os.environ.get("MY_API_KEY", "Key not available!")
   ```

### Changing the Schedule
Edit the `cron` expression in `.github/workflows/actions.yml`:
```yaml
on:
  schedule:
    - cron: '0 0 * * 1'  # Every Monday at 00:00 UTC
```
Use [crontab.guru](https://crontab.guru) to build cron expressions. An hourly alternative is already commented out in the file.

### Adding Dependencies
Add packages to `requirements.txt` with pinned versions:
```
requests==2.28.1
some-package==1.2.3
```

## Conventions

- **Python version**: 3.11 (pinned in `.python-version` and `actions.yml`)
- **Logging**: Use the module-level `logger` from `main.py`; do not use `print()` for output
- **Log format**: `%(asctime)s - %(name)s - %(levelname)s - %(message)s`
- **Log file**: `status.log` — rotating, 1 MB max with 1 backup file; this file is committed back to the repo by the Action after each run
- **Secrets**: Never hardcode secrets; always read from `os.environ` and handle `KeyError` gracefully
- **Workflow bot identity**: Commits made by the Action use `action@github.com` / `GitHub Action`

## GitHub Actions Notes

- The workflow only runs on the `main` branch push-back step; any changes committed by the Action land on `main`
- The `git diff-index --quiet HEAD || (git commit ...)` pattern means a commit only occurs when there are actual file changes (e.g., `status.log` was updated)
- `GITHUB_TOKEN` is automatically provided by GitHub — no manual secret needed for the push step

## Extending the Template

Common customizations:
- **Different API**: Replace the weather fetch in `main.py` with any HTTP request
- **Data persistence**: Write results to a CSV/JSON file alongside `status.log`; both will be committed automatically
- **Notifications**: Add a step to send a Slack/email notification on failure using `if: failure()` in the workflow
- **Multiple scripts**: Add additional `run: python other_script.py` steps in the workflow
- **Disable auto-commit**: Remove the "commit files" and "push changes" steps if log persistence in the repo is not needed
