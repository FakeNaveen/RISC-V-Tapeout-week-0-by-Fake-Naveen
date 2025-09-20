# RISC-V-Tapeout-week-0-by-Fake-Naveen
A collaborative initiative by VSD and IIT Gandhinagar, the RISC-V SoC Tapeout Program offers a hands-on journey through SoC design and tapeout using the RISC-V ISA. Participants learn RTL, simulation, synthesis, physical design, and verification, bridging academics with real chip design.

# RISC-V SoC Tapeout Program
**Collaborative Program** — VSD × IIT Gandhinagar  
**Week 0 Deliverable:** GitHub repository + tool installation & verification

---

## Table of Contents
1. [Introduction](#introduction)  
2. [Goals & Scope](#goals--scope)  
3. [System Requirements & Recommended VM](#system-requirements--recommended-vm)  
4. [Tools Overview & Purpose](#tools-overview--purpose)  
5. [Quick install scripts](#quick-install-scripts)  
6. [Manual installation instructions (by tool)](#manual-installation-instructions-by-tool)  
7. [Verification / Smoke tests](#verification--smoke-tests)  
8. [Repository layout & files added](#repository-layout--files-added)  
9. [Usage examples / Typical workflow](#usage-examples--typical-workflow)  
10. [Troubleshooting & Notes](#troubleshooting--notes)  
11. [Acknowledgements & References](#acknowledgements--references)  
12. [Conclusion](#conclusion)

---

## Introduction
This repository is the Week-0 deliverable for the **RISC-V SoC Tapeout program**, created for documenting the environment setup and providing automated installation scripts for the EDA and simulation toolchain needed during the program. The goal is to make onboarding reproducible, auditable, and easy to share with mentors and peers.

---

## Goals & Scope
- Create a GitHub repo describing the program and environment.
- Provide step-by-step installation instructions for all required tools.
- Offer automation (scripts) to reduce setup friction.
- Provide verification steps so you can confirm a working environment.

---

## System Requirements & Recommended VM
Minimum recommended configuration (per program guidelines):
- **RAM:** 6 GB (8+ GB recommended)  
- **Disk:** 50 GB free (more if you keep multiple PDKs)  
- **CPU:** 4 vCPU  
- **OS:** Ubuntu 20.04 LTS or later (Ubuntu 20.04+ recommended)  
- **Recommended VM:** Oracle VirtualBox (host: Windows/macOS/Linux) — download from https://www.virtualbox.org/wiki/Downloads

> Note: these requirements and tool choices align with the provided installation guidelines. :contentReference[oaicite:1]{index=1}

---

## Tools Overview & Purpose
Below are the tools used in this program, brief descriptions of why they’re required, and where they fit in the flow.

- **Yosys** — RTL synthesis tool (converts Verilog to gate-level netlists). Essential for flow from RTL to physical design.
- **Icarus Verilog (iverilog)** — Verilog simulator for running testbenches; useful for pre-synthesis functional verification.
- **GTKWave** — Waveform viewer for viewing simulation VCD/FSDB outputs.
- **ngspice** — Circuit simulator (analog SPICE). Useful when analog macros or mixed-signal elements are present.
- **magic** — Layout tool used for final layout verification and layout editing (used with LVS/DRC).
- **OpenLane** — Automated digital ASIC flow (combines synthesis, place & route, LVS/DRC integration with PDKs and OpenROAD tools); core to crate GDS for tapeout.
- **Docker** — Container runtime used to run some parts of the flow reproducibly; OpenLane uses Docker images heavily.
- **(Optional) OpenSTA** — Static timing analysis tool (guideline mentions it's not required for SFAL participants but is part of the ecosystem).

These tools and their installation steps are adapted from the program installation document. :contentReference[oaicite:2]{index=2}

---

## Quick install scripts
Below is a single script you can run (on a fresh Ubuntu 20.04+ VM) to install most of the tools. **Run as a user with sudo privileges.**

> Save as `install_scripts/install_all.sh` and run:
> `chmod +x install_scripts/install_all.sh && ./install_scripts/install_all.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

echo "Updating apt..."
sudo apt-get update -y
sudo apt-get upgrade -y

echo "Installing common dependencies..."
sudo apt-get install -y build-essential git curl wget \
    python3 python3-venv python3-pip make pkg-config \
    bison flex libreadline-dev gawk tcl-dev libffi-dev \
    graphviz xdot libboost-system-dev libboost-python-dev \
    libboost-filesystem-dev zlib1g-dev m4 tcsh csh \
    libx11-dev tcl-dev tk-dev libcairo2-dev mesa-common-dev \
    libglu1-mesa-dev libncurses-dev apt-transport-https \
    ca-certificates software-properties-common

# Install Yosys
if ! command -v yosys >/dev/null 2>&1; then
  echo "Installing Yosys..."
  git clone https://github.com/YosysHQ/yosys.git /tmp/yosys
  cd /tmp/yosys
  sudo apt-get install -y clang
  make config-gcc
  make -j"$(nproc)"
  sudo make install
fi

# Install Icarus Verilog
echo "Installing iverilog..."
sudo apt-get install -y iverilog

# Install GTKWave
echo "Installing gtkwave..."
sudo apt-get install -y gtkwave

# Install ngspice (from source recommended)
echo "Installing ngspice..."
NGSPICE_VERSION="ngspice-37"
cd /tmp
if [ ! -d "$NGSPICE_VERSION" ]; then
  # You may need to replace with the exact tarball name you downloaded
  wget -q "https://sourceforge.net/projects/ngspice/files/ng-spice-rework/37/$NGSPICE_VERSION.tar.gz/download" -O ngspice.tar.gz
  tar -zxvf ngspice.tar.gz
  cd "$NGSPICE_VERSION"
  mkdir -p release && cd release
  ../configure --with-x --with-readline=yes --disable-debug
  make -j"$(nproc)"
  sudo make install
fi

# Install Magic
echo "Installing Magic..."
cd /tmp
if [ ! -d magic ]; then
  git clone https://github.com/RTimothyEdwards/magic.git
  cd magic
  ./configure
  make -j"$(nproc)"
  sudo make install
fi

# Install Docker (required for OpenLane)
echo "Installing Docker..."
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo groupadd -f docker
sudo usermod -aG docker "$USER"

# Install OpenLane (cloned repo + make)
echo "Installing OpenLane..."
cd "$HOME"
if [ ! -d OpenLane ]; then
  git clone https://github.com/The-OpenROAD-Project/OpenLane.git
  cd OpenLane
  make -j"$(nproc)"
  make test || echo "OpenLane tests had issues; continue to manual checks"
fi

echo "Installation script complete. Reboot may be required for docker group membership to take effect."

