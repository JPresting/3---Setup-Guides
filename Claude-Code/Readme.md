# Claude Code Installation on Windows

Quick setup guide for installing Claude Code on Windows using WSL.

## Prerequisites
- Windows 10/11 with WSL support

## Installation Steps

1. **Install WSL with Ubuntu**
   ```powershell
   wsl --install -d Ubuntu
   ```

2. **Setup Ubuntu user account**
   - Create username and password when prompted

3. **Update system and install Node.js**
   ```bash
   sudo apt update && sudo apt upgrade -y
   curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```

4. **Install Claude Code**
   ```bash
   sudo npm install -g @anthropic-ai/claude-code
   ```

5. **Fix command alias (important!)**
   ```bash
   echo 'alias claude-code="node /usr/lib/node_modules/@anthropic-ai/claude-code/cli.js"' >> ~/.bashrc
   source ~/.bashrc
   ```

## Usage

1. Open Ubuntu terminal (search "Ubuntu" in Windows start menu)
2. Run:
   ```bash
   claude-code
   ```

## First Run Setup
- Choose theme preference
- Select login method (Claude subscription or API key)
- Complete authentication

That's it! Claude Code is now ready to use.