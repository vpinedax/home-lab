# Home Lab — Self-Hosted Infrastructure
 
A personal home lab running on a **Beelink S12 Mini PC** (Intel N100, 8GB RAM, 256GB SSD) with YAML configuration files and Docker containers.
 
## Setup
 
| Layer | Technology |
|---|---|
| Hypervisor | Proxmox VE |
| VM | Home Assistant OS |
| Container Runtime | Docker (inside LXC) |
| NVR / AI Detection | Frigate (inside Docker) |
| Automation | Home Assistant |
| Hardware Acceleration | Intel iGPU (VAAPI / OpenVINO) |
 
## Architecture
 
```
Beelink S12 (Proxmox)
├── VM: Home Assistant (hoass)
│   └── Frigate Integration
└── LXC: Docker
    └── Frigate NVR config (MQTT, go2rtc, nginx)
```
 
## Cameras
 
- **Reolink Doorbell** — two-way audio, HTTP-FLV main stream, RTSP sub for detection
- **Amcrest (Front)** — dual-stream RTSP with audio
- **Amcrest (Kitchen)** — dual-stream RTSP with audio
  
## AI Features
 
- Object detection via **OpenVINO** on Intel iGPU (offloaded from CPU)
- Frigate Plus model for enhanced detection of delivery packages and vehicles (Amazon, FedEx, UPS, USPS)
- Custom pet recognition (sub-label classification)
- Bird classification
- Face recognition
- Detection zones with per-zone object filtering (front yard, driveway, parking spot, street)
## Repo Structure
 
```
home-lab/       
├── homeassistant/
│   └── configuration.yaml 
├── docker/
│   └── docker-compose.yml
|   └── frigate_config.yml
├── .env.example           
└── README.md
```
 
## Hardware
 
| Component | Spec |
|---|---|
| Device | Beelink S12 Mini PC |
| CPU | Intel N100 (4-core) |
| RAM | 8GB DDR4 |
| Storage | 256GB NVMe SSD |
| GPU | Intel UHD Graphics (iGPU) |
