# Agent Skills Repository

This repository contains professional agent skills for AI-powered coding assistants (Junie, Claude Code, Cursor, etc.).

## Skills Included

### 📦 [spmForKmp](./spmforkmp)
A comprehensive skill for working with the **spmForKmp** Gradle plugin (`io.github.frankois944.spmForKmp`).
- Simplifies Swift Package Manager (SPM) integration in Kotlin Multiplatform projects.
- Provides guidance on Swift bridge code, cinterop, and Objective-C compatibility.
- Includes a strict "Definition of Done" to ensure successful builds and simulator launches.

## Installation

### Using `npx skills` (Recommended)
You can install the `spmforkmp` skill globally or for a specific project using the Agent Skills CLI:

**Project-level:**
```bash
npx skills add frankois944/skills/spmforkmp
```

**Global:**
```bash
npx skills add -g frankois944/skills/spmforkmp
```

### Updating the Skill
If you have already installed the skill, you can update it to the latest version by running:

```bash
npx skills update spmforkmp
```

### Manual Installation for Junie

**Project-level:**
```bash
mkdir -p .junie/skills
git clone https://github.com/frankois944/skills.git temp_skills
cp -r temp_skills/spmforkmp .junie/skills/
rm -rf temp_skills
```

**Global:**
```bash
mkdir -p ~/.junie/skills
git clone https://github.com/frankois944/skills.git temp_skills
cp -r temp_skills/spmforkmp ~/.junie/skills/
rm -rf temp_skills
```

## Contributing
Feel free to open issues or pull requests to improve these skills.

## License
[MIT](./LICENSE)
