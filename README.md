
## 🧱 Architecture Overview

### 1. **Host System**

* **Raspberry Pi 4** (or compatible SBC)
* **Operating System**: Raspberry Pi OS (64-bit recommended)
* **Containerization**: Docker or Podman

### 2. **Core Components**

* **pigpio Daemon**: Manages GPIO access across containers
* **Docker Containers**: Each emulates an ESP32 node
* **GPIO Pin Mapping**: Configured per node to avoid conflicts

---

## 🛠️ Step-by-Step Implementation

### 1. **Install pigpio Daemon on Host**

```bash
sudo apt update
sudo apt install pigpio
sudo systemctl enable pigpiod
sudo systemctl start pigpiod
```

* The `pigpiod` daemon allows containers to access GPIO pins over TCP.

### 2. **Configure GPIO Pin Mapping**

* Create a configuration file (e.g., `gpio_mapping.json`) to define which physical GPIO pins correspond to each virtual ESP32 node.

```json
{
  "node1": [17, 27],
  "node2": [22, 23]
}
```

* This ensures each container uses distinct GPIO pins, preventing conflicts.

### 3. **Set Up Docker Container for Each Node**

* **Dockerfile**: Base image with Python and `pigpio` library.

```Dockerfile
FROM python:3.9-slim
RUN pip install pigpio
COPY node_script.py /app/node_script.py
CMD ["python", "/app/node_script.py"]
```

* **node_script.py**: Simulates ESP32 behavior, reads/writes GPIO pins.

```python
import pigpio
import time

pi = pigpio.pi("host_ip")  # Replace with host's IP address
if not pi.connected:
    exit()

# Define GPIO pins from config
gpio_pins = [17, 27]  # Example for node1

# Set pins as outputs
for pin in gpio_pins:
    pi.set_mode(pin, pigpio.OUTPUT)

# Blink LEDs
while True:
    for pin in gpio_pins:
        pi.write(pin, 1)
    time.sleep(1)
    for pin in gpio_pins:
        pi.write(pin, 0)
    time.sleep(1)
```

* **Build and Run**:

```bash
docker build -t esp32_node .
docker run --net=host --device=/dev/gpiomem esp32_node
```

* The `--net=host` flag allows the container to use the host's network, and `--device=/dev/gpiomem` grants access to GPIO memory.

### 4. **Networking Between Containers**

* Use Docker's bridge network to enable communication between containers.

```bash
docker network create esp_network
docker run --net=esp_network --name=esp32_node1 esp32_node
docker run --net=esp_network --name=esp32_node2 esp32_node
```

* Containers can now communicate over the `esp_network`, simulating ESP-Now or BLE interactions.

---

## 🔄 Alternative: Using Renode for Emulation

For more advanced emulation, consider using **Renode**, which supports GPIO simulation and can emulate multiple ESP32 nodes.

* **Renode Setup**:

```bash
# Clone Renode repository
git clone https://github.com/renode/renode.git
cd renode
# Follow Renode's documentation to set up ESP32 emulation
```

* **GPIO Simulation**:

  * Renode allows creating GPIO connectors between virtual machines.
  * Example:

  ```renode
  emulation CreateGPIOConnector "gpio-con"
  machine LoadPlatformDescription @path/to/esp32.repl
  machine LoadPlatformDescription @path/to/esp32.repl
  gpio-con AddMachine gpio-con 0 7
  gpio-con AddMachine gpio-con 1 4
  ```

  * This connects GPIO pin 7 of the first machine to GPIO pin 4 of the second machine.

---

## ✅ Final Thoughts

This setup provides a scalable solution for emulating multiple ESP32 nodes with real GPIO interaction on a Raspberry Pi. It's ideal for:

