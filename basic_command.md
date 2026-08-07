## Node.js Installation

Install **Node.js LTS** using **NVM (Node Version Manager)**.

### 1. Install NVM

Run the following command:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

### 2. Reload your shell configuration

```bash
source ~/.bashrc
```

> If you're using a different shell, such as Zsh, reload the appropriate configuration file.

### 3. Install the latest Node.js LTS version

```bash
nvm install --lts
```

### 4. Verify the installation

```bash
node --version
npm --version
```

You should now have the latest **Node.js LTS** version and its bundled **npm** installed.
