### BWH Hive

Modern Project Management Software

### Installation

You can install this app using the [bench](https://github.com/frappe/bench) CLI:

```bash
cd $PATH_TO_YOUR_BENCH
bench get-app $URL_OF_THIS_REPO --branch develop
bench install-app bwh_hive
```

### Contributing

This app uses `pre-commit` for code formatting and linting. Please [install pre-commit](https://pre-commit.com/#installation) and enable it for this repository:

```bash
cd apps/bwh_hive
pre-commit install
```

Pre-commit is configured to use the following tools for checking and formatting your code:

- ruff
- eslint
- prettier
- pyupgrade
### CI

This app can use GitHub Actions for CI. The following workflows are configured:

- CI: Installs this app and runs unit tests on every push to `develop` branch.
- Linters: Runs [Frappe Semgrep Rules](https://github.com/frappe/semgrep-rules) and [pip-audit](https://pypi.org/project/pip-audit/) on every pull request.


### License

agpl-3.0

### FAQ

**What is BWH Hive?**

BWH Hive (`bwh_hive`) is a Modern Project Management Software built on the [Frappe](https://github.com/frappe/frappe) framework. It provides a React-based frontend and a Python backend for managing projects, tasks, and team collaboration.

**How do I run it locally?**

Install the app with `bench` (see Installation above), then start the Frappe backend:

```bash
bench start
```

For frontend development, run the Vite dev server separately:

```bash
cd apps/bwh_hive/frontend
yarn dev
```

The Frappe backend is available at `pms.localhost:8000` and the frontend dev server at `localhost:8080`.

**How do I contribute?**

Fork the repository, create a feature branch off `develop`, and run `pre-commit install` in the app directory before committing (see Contributing above). Open a pull request targeting the `develop` branch.
