# GB2 Terminal Monorepo

This is a container monorepo for related GB2 Terminal projects. Each project is maintained as an independent git submodule, allowing them to be developed standalone while providing AI coding assistants with complete context across all related projects.

## Projects

### 🌐 gb2-terminal-web
Web application built with React, TypeScript, and Vite.

**Location:** `./gb2-terminal-web`  
**Repository:** [gb2-terminal-web](https://github.com/caretapps-developer/gb2-terminal-web.git)

### 📱 gb2-terminal-expo
Mobile application built with React Native and Expo.

**Location:** `./gb2-terminal-expo`  
**Repository:** [gb2-terminal-expo](https://github.com/caretapps-developer/gb2-terminal-expo.git)

## Getting Started

### Initial Clone

When cloning this repository for the first time, you need to initialize and update the submodules:

```bash
git clone <this-repo-url>
cd gb2-terminal-monorepo
git submodule init
git submodule update
```

Or clone with submodules in one command:

```bash
git clone --recurse-submodules <this-repo-url>
```

### Updating Submodules

To pull the latest changes from all submodules:

```bash
git submodule update --remote --merge
```

To update a specific submodule:

```bash
cd gb2-terminal-web  # or gb2-terminal-expo
git pull origin main
cd ..
git add gb2-terminal-web
git commit -m "Update gb2-terminal-web submodule"
```

## Working with Submodules

### Making Changes in a Submodule

Each submodule is a standalone git repository. To make changes:

1. Navigate into the submodule directory:
   ```bash
   cd gb2-terminal-web
   ```

2. Make your changes and commit them:
   ```bash
   git add .
   git commit -m "Your commit message"
   git push origin main
   ```

3. Return to the monorepo root and update the submodule reference:
   ```bash
   cd ..
   git add gb2-terminal-web
   git commit -m "Update gb2-terminal-web submodule reference"
   git push
   ```

### Adding a New Submodule

To add another related project:

```bash
git submodule add <repository-url> <directory-name>
git commit -m "Add <project-name> submodule"
```

### Removing a Submodule

If you need to remove a submodule:

```bash
git submodule deinit -f <submodule-path>
git rm -f <submodule-path>
rm -rf .git/modules/<submodule-path>
git commit -m "Remove <submodule-name> submodule"
```

## Project Structure

```
gb2-terminal-monorepo/
├── .gitmodules              # Submodule configuration
├── .gitignore              # Root-level gitignore
├── README.md               # This file
├── gb2-terminal-web/       # Web application submodule
└── gb2-terminal-expo/      # Mobile application submodule
```

## Benefits of This Setup

1. **Standalone Projects**: Each project maintains its own git history and can be developed independently
2. **Unified Context**: AI coding assistants can access code from all related projects for better context
3. **Version Control**: The monorepo tracks specific commits of each submodule
4. **Flexible Development**: Work on projects individually or together as needed
5. **Independent Deployment**: Each project can be deployed separately

## Development Workflow

### For Web Development
```bash
cd gb2-terminal-web
npm install
npm run dev
```

### For Mobile Development
```bash
cd gb2-terminal-expo
npm install
npm start
```

## Notes

- Each submodule points to a specific commit in its repository
- When you pull changes in the monorepo, submodules won't automatically update
- Always commit submodule changes in the submodule repository first, then update the reference in the monorepo
- The `.idea/` directory is ignored at the root level for JetBrains IDEs

## Resources

- [Git Submodules Documentation](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [Working with Submodules](https://github.blog/2016-02-01-working-with-submodules/)

