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
8. [Acknowledgements & References](#acknowledgements--references)  
9. [Conclusion](#conclusion)

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
```

---

## Manual Installation Instructions (By Tool)

Designing a chip is like preparing a symphony — each tool is an instrument with its own role, and when they play together, we get silicon as the final masterpiece. Below are the manual installation steps for each tool, along with why they matter.

🛠️ Yosys – The Composer (Synthesis)

Think of Yosys as the composer of our design symphony. It takes the high-level music (Verilog RTL) and translates it into precise sheet notes (gate-level netlists) that other players can understand.

Install:
```bash
sudo apt-get update
git clone https://github.com/YosysHQ/yosys.git
cd yosys
sudo apt install -y make clang build-essential bison flex \
  libreadline-dev gawk tcl-dev libffi-dev git \
  graphviz xdot pkg-config python3 \
  libboost-system-dev libboost-python-dev libboost-filesystem-dev zlib1g-dev
make config-gcc
make -j$(nproc)
sudo make install
```

🖥️ Icarus Verilog – The Performer (Simulation)

Before we send our design to fabrication, we need to see if it performs correctly. Icarus Verilog plays the RTL as a musician would, ensuring the notes sound right.

Install:

```
sudo apt-get update
sudo apt-get install -y iverilog
```

🌊 GTKWave – The Conductor’s Score (Waveform Viewer)

When music is played, the conductor sees the waves of sound. GTKWave lets us “see” our Verilog simulations as waveforms, so we can spot bugs like sour notes in a melody.

Install:

```
sudo apt-get update
sudo apt-get install -y gtkwave
```

🔌 ngspice – The Sound Engineer (Analog Simulation)

Not all sounds are digital — some are analog. ngspice is our sound engineer, testing circuits at the transistor level and ensuring analog macros (like PLLs) blend smoothly with digital design.

Install:

```
tar -zxvf ngspice-37.tar.gz
cd ngspice-37
mkdir release && cd release
../configure --with-x --with-readline=yes --disable-debug
make -j$(nproc)
sudo make install
```

🧩 Magic – The Architect (Layout Tool)

Once the music is written, we need a concert hall. Magic is the architect of our chip layouts, letting us see and edit the physical placement of transistors and wires. It checks whether everything is built safely and to spec.

Install:

```
sudo apt-get install -y m4 tcsh csh libx11-dev tcl-dev tk-dev \
  libcairo2-dev mesa-common-dev libglu1-mesa-dev libncurses-dev
git clone https://github.com/RTimothyEdwards/magic
cd magic
./configure
make -j$(nproc)
sudo make install
```

📦 Docker – The Stage Manager (Environment Control)

Concerts fall apart without logistics. Docker ensures every tool and dependency is packed neatly in containers, so the show runs identically on every machine.

Install:

```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo groupadd -f docker
sudo usermod -aG docker $USER
# Logout and login again or reboot
```

🚀 OpenLane – The Orchestra (RTL to GDS Flow)

Finally, OpenLane brings together all instruments — from synthesis to layout — into one orchestra. It automates the RTL-to-GDSII flow so your design can be ready for tapeout.

Install:

```
cd $HOME
git clone https://github.com/The-OpenROAD-Project/OpenLane
cd OpenLane
make
make test
```


🔑 Tip: Installing manually deepens understanding — you see what each tool needs, how they interconnect, and why order matters. Together, these tools create the complete ecosystem required to take your design from concept to silicon reality.

---

## Verification / Smoke Tests

After installing all tools, it’s important to confirm that they are working correctly. Below are simple version checks and small test runs to ensure your environment is set up properly.

🔎 Version Checks

Run these commands to confirm the tools are installed and accessible:

```
yosys -V           # Yosys version
iverilog -v        # Icarus Verilog version
gtkwave --version  # GTKWave version
ngspice -v         # ngspice version
magic -v           # Magic version
docker --version   # Docker version
python3 --version  # Python version
git --version      # Git version
```

🧪 Functional Test – Verilog Simulation

Create a simple Verilog file and simulate it with Icarus Verilog:

```
cat > hello.v <<'EOF'
module tb;
  initial begin
    $display("Hello, RISC-V SoC Tapeout!");
    $finish;
  end
endmodule
EOF

iverilog -o hello_tb.vvp hello.v
vvp hello_tb.vvp
```

Expected output:

```
Hello, RISC-V SoC Tapeout!
```

📈 Waveform Check with GTKWave

Modify the testbench to dump waveforms:

```
cat > hello_wave.v <<'EOF'
module tb;
  initial begin
    $dumpfile("hello.vcd");
    $dumpvars(0, tb);
    #10 $finish;
  end
endmodule
EOF

iverilog -o hello_wave.vvp hello_wave.v
vvp hello_wave.vvp
gtkwave hello.vcd &
```


GTKWave should open and display the timeline of signals.

⚡ OpenLane Quick Test

If OpenLane was installed with Docker, run the included test flow:

```
cd ~/OpenLane
make test
```


If the setup is correct, you will see logs indicating the synthesis, placement, and routing of a small design.

---


## Acknowledgements & References
🙏 Acknowledgements

VLSI System Design (VSD)
For their pivotal role in co-developing the RISC-V SoC Tapeout Program, providing industry-grade tools, and facilitating hands-on training. 
VLSI System Design

Indian Institute of Technology Gandhinagar (IITGN)
For hosting and executing the program, offering access to state-of-the-art infrastructure, and fostering a collaborative research environment. 
IIT Gandhinagar

Synopsys
For supplying Electronic Design Automation (EDA) tools essential for the design and verification processes.

SCL180 PDK
For providing the Process Design Kit that enabled the physical design and tapeout stages.

Participating Students and Faculty
For their dedication, innovative contributions, and collaborative spirit throughout the program.

📚 References

VLSI System Design (VSD)
Official website detailing the RISC-V Reference SoC Tapeout Program:

VLSI System Design

Indian Institute of Technology Gandhinagar (IITGN)
Information on the Visiting Students Programme:

IIT Gandhinagar

Synopsys
Provider of EDA tools for chip design and verification.

SCL180 PDK
Process Design Kit used for the physical design and tapeout stages.

---

## Conclusion
🎯 Program Overview

The RISC-V SoC Tapeout Program is a comprehensive 20-week initiative developed collaboratively by VLSI System Design (VSD) and Indian Institute of Technology Gandhinagar (IITGN). This program offers participants a hands-on experience in designing, implementing, and fabricating a RISC-V System-on-Chip (SoC) using industry-grade tools and methodologies. 
VLSI System Design

🛠️ Key Highlights

End-to-End Design Flow: Participants engage in the complete SoC design cycle, from RTL design to GDSII and post-silicon validation.

Industry-Grade Tools: Utilization of Synopsys EDA tools and the SCL180 nm Process Design Kit (PDK) ensures a realistic design environment.

Collaborative Learning: The program fosters collaboration among students, educators, and professionals, promoting knowledge sharing and innovation.

Real-World Application: The program aligns with India's Semiconductor Mission, aiming to develop skilled silicon designers capable of contributing to the nation's semiconductor ecosystem.

# work by : Naveen J
