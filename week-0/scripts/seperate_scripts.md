## Manual Installation Instructions (By Tool)

Designing a chip is like preparing a symphony — each tool is an instrument with its own role, and when they play together, we get silicon as the final masterpiece. Below are the manual installation steps for each tool, along with why they matter.

# 🛠️ Yosys – The Composer (Synthesis)

Think of Yosys as the composer of our design symphony. It takes the high-level music (Verilog RTL) and translates it into precise sheet notes (gate-level netlists) that other players can understand.

Install:

```
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

# 🖥️ Icarus Verilog – The Performer (Simulation)

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

# 🔌 ngspice – The Sound Engineer (Analog Simulation)

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

# 🧩 Magic – The Architect (Layout Tool)

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

# 📦 Docker – The Stage Manager (Environment Control)

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

# 🚀 OpenLane – The Orchestra (RTL to GDS Flow)

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
