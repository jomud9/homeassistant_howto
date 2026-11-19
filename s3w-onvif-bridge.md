# Sonoff CAM-S3W ONVIF Bridge for UniFi Protect (`s3w-onvif-bridge`) Installation Guide

This guide rebuilds the `s3w-onvif-bridge` Debian LXC container from scratch. All network, camera, stream, and snapshot settings are centralized in `/etc/onvif_bridge.conf`, and `/opt/mediamtx.yml` is auto-generated from that configuration whenever `mediamtx.service` starts.

Although the Sonoff CAM-S3W advertises ONVIF and RTSP support, its non-standard ONVIF implementation—particularly around WS-Security digest authentication and SOAP schema quirks common on budget cameras—causes UniFi Protect to reject direct adoption with an "Invalid Credentials" error. This proxy sits in the middle to pull the camera's working RTSP feeds directly and expose a clean, standards-compliant ONVIF endpoint with dual HQ/Low profiles and snapshots that UniFi Protect can adopt reliably.


Changlog 20261119 - Initial github version

---
---

## Disclaimer

> **Use at Your Own Risk:** I take no credit and/or responsibility for the code provided in this guide, as it was generated with the assistance of Google Gemini. However, I have personally tested and verified this exact setup on my own environment:
> * **Hypervisor:** Proxmox VE 9.2.28
> * **Guest OS:** Debian 13 (Trixie) LXC
> * **NVR:** UniFi Protect v7.3.74
> * **Camera:** Sonoff CAM-S3W (Firmware `1.2.1`)
>
> However feel free to ping me on GitHub if you run into questions or need help!

---

## Architecture & `/opt/onvif_bridge.py` Logic Overview

```text
+---------------------------------------------------------------------------------------------------+
|                                   s3w-obridge LXC (10.15.30.242)                                  |
|                                                                                                   |
|  +---------------------------+                                                                    |
|  |  /etc/onvif_bridge.conf   |  (Single Source of Truth: IPs, Credentials, Streams, Snapshot)     |
|  +-------------+-------------+                                                                    |
|                |                                                                                  |
|                +---> [1. Boot / ExecStartPre] (--generate-mediamtx)                               |
|                |        Writes /opt/mediamtx.yml                                                  |
|                |                   |                                                              |
|                |                   v                                                              |
|                |     +---------------------------+       TCP RTSP Pull       +-----------------+  |
|                |     |     mediamtx.service      | <------------------------ |  Sonoff CAM-S3W |  |
|                |     |       (Port 8554)         |  ch0 (2304x1296)          |  (10.15.30.241) |  |
|                |     |                           |  ch1 (640x360)            |   (Port 554)    |  |
|                |     +------+-------------+------+                           +-----------------+  |
|                |            |             |                                                       |
|                |            |             +--- Local RTSP (127.0.0.1:8554/sonoff_sub)             |
|                |            |                  |                                                  |
|                +---> [2. Runtime Daemon]       v                                                  |
|                      +---------------------------------------+                                    |
|                      |        onvif-bridge.service           |                                    |
|                      |     (/opt/onvif_bridge.py :8000)      |                                    |
|                      |                                       |                                    |
|                      |  * Background Thread (every 30s)      |                                    |
|                      |    ffmpeg -> /tmp/sonoff_snap.jpg     |                                    |
|                      |                                       |                                    |
|                      |  * HTTP/SOAP Handler (Port 8000)      |                                    |
|                      |    - POST /onvif/* -> SOAP XML        |                                    |
|                      |      (Profiles, Stream & Snap URIs)   |                                    |
|                      |    - GET /snapshot.jpg -> Cached JPEG |                                    |
|                      +-------------------+-------------------+                                    |
|                                          |                                                        |
+------------------------------------------|--------------------------------------------------------+
                                           |
                       +-------------------+-------------------+
                       |                                       |
         ONVIF SOAP & JPEG (:8000)               RTSP Video Streams (:8554)
                       |                                       |
                       v                                       v
           +-------------------------------------------------------+
           |                     UniFi Protect                     |
           |             (Adopts 10.15.30.242:8000)                |
           +-------------------------------------------------------+
```



## 0. Get a Sonoff S3W Camera, and set it up
> This is how your nightmare start   :P

Follow all instruction via your phone app. 

1. Get it connected your wifi. Reserve that with a static IP address ( `10.15.30.241` for example )
1. Go to Web admin page http://\<YOUR\_CAM\_IP\> ( `http://10.15.30.241` ) For your first login you need to assign your password for admin page access
1. You have fun with all options, be my guest.
1. Go `Device Settings` > `ONVIF/RTSP` > Switch to `ON`
1. Assign `ONVIF/RTSP Username` (`sonoff`)
1. Assign `ONVIF/RTSP Password` (`camfeed`)

We will need these information on Step 4.

---

## 1. Create a Proxmox LXC Container

Create an unprivileged Debian LXC container (hostname `s3w-obridge` for this case) with the following specifications:

| Resource | Recommended Value | Notes |
| :--- | :--- | :--- |
| **OS Template** | Debian 13 (Trixie) | Standard LXC template |
| **Type** | Unprivileged (`unprivileged: 1`) | |
| **CPU** | 1 vCPU (resize if needed) | Uses ~1–2% CPU at runtime |
| **Memory / Swap** | 128 MiB RAM / 128 MiB Swap (resize if needed) | Uses ~38 MiB RAM at runtime |
| **Boot Disk** | 4 GiB | Uses ~1.0 GiB total |
| **Network (`net0`)** | `10.15.30.242/24`, GW `10.15.30.1` | Pin via static or DHCP reservation |

---

## 2. Initial OS Configuration & Dependencies

Open the LXC console as `root`, set the timezone to Hong Kong (`HKT`), update the system, and install required packages:

```bash
timedatectl set-timezone Asia/Hong_Kong
apt update && apt upgrade -y
apt install -y python3 ffmpeg curl tar ca-certificates
```

**Verification:**
`timedatectl`
Confirm `Time zone:` reports `Asia/Hong_Kong (HKT, +0800)`

---

## 3. Install MediaMTX Binary

Download and extract the latest `amd64` release of MediaMTX into `/opt`:

```bash
cd /opt
MTX_VER=$(curl -s https://api.github.com/repos/bluenviron/mediamtx/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
curl -L "https://github.com/bluenviron/mediamtx/releases/download/${MTX_VER}/mediamtx_${MTX_VER}_linux_amd64.tar.gz" -o mediamtx.tar.gz
tar -xzf mediamtx.tar.gz mediamtx
rm -f mediamtx.tar.gz
chmod +x /opt/mediamtx
```

**Verification:**
`/opt/mediamtx --version
`

---

## 4. Central Configuration File (`/etc/onvif_bridge.conf`)

Create `/etc/onvif_bridge.conf` to store all host IPs, camera credentials, RTSP stream paths, and snapshot parameters in one file:

```bash
cat << 'EOF' > /etc/onvif_bridge.conf
[network]
host_ip = 10.15.30.242   # Your IP in LXC
onvif_port = 8000
rtsp_port = 8554

[camera]
ip = 10.15.30.241        # Your WebCam IP Address
rtsp_port = 554
manufacturer = SONOFF
model = CAM-S3W
firmware = 1.0.0
serial = S3W001
hardware_id = 1.0
username = sonoff        # Your ONVIF/RTSP username
password = camfeed		 # Your ONVIF/RTPS password

[streams]
main_path = sonoff
main_source_path = av_stream/ch0
main_width = 2304
main_height = 1296

sub_path = sonoff_sub
sub_source_path = av_stream/ch1
sub_width = 640
sub_height = 360

[snapshot]
enabled = true
refresh_interval = 30
cache_path = /tmp/sonoff_snap.jpg
EOF

chmod 600 /etc/onvif_bridge.conf
```

---

## 5. ONVIF Bridge & Config Generator (`/opt/onvif_bridge.py`)

Create `/opt/onvif_bridge.py`. It reads `/etc/onvif_bridge.conf`, supports `--generate-mediamtx` to render `/opt/mediamtx.yml` before MediaMTX starts, serves ONVIF SOAP endpoints on port `8000`, and refreshes `/tmp/sonoff_snap.jpg` every 30 seconds:

```bash
cat << 'EOF' > /opt/onvif_bridge.py
import configparser
import http.server
import os
import socketserver
import subprocess
import sys
import threading
import time

CONFIG_PATH = "/etc/onvif_bridge.conf"
MEDIAMTX_YML_PATH = "/opt/mediamtx.yml"

cfg = configparser.ConfigParser()
cfg.read(CONFIG_PATH)

HOST_IP = cfg.get("network", "host_ip", fallback="10.15.30.242")
ONVIF_PORT = cfg.getint("network", "onvif_port", fallback=8000)
RTSP_PORT = cfg.getint("network", "rtsp_port", fallback=8554)

CAM_IP = cfg.get("camera", "ip", fallback="10.15.30.241")
CAM_RTSP_PORT = cfg.getint("camera", "rtsp_port", fallback=554)
CAM_USER = cfg.get("camera", "username", fallback="sonoff")
CAM_PASS = cfg.get("camera", "password", fallback="camfeed")
MANUFACTURER = cfg.get("camera", "manufacturer", fallback="SONOFF")
MODEL = cfg.get("camera", "model", fallback="CAM-S3W")
FIRMWARE = cfg.get("camera", "firmware", fallback="1.0.0")
SERIAL = cfg.get("camera", "serial", fallback="S3W001")
HARDWARE_ID = cfg.get("camera", "hardware_id", fallback="1.0")

MAIN_PATH = cfg.get("streams", "main_path", fallback="sonoff")
MAIN_SOURCE_PATH = cfg.get("streams", "main_source_path", fallback="av_stream/ch0").lstrip("/")
MAIN_WIDTH = cfg.getint("streams", "main_width", fallback=2304)
MAIN_HEIGHT = cfg.getint("streams", "main_height", fallback=1296)

SUB_PATH = cfg.get("streams", "sub_path", fallback="sonoff_sub")
SUB_SOURCE_PATH = cfg.get("streams", "sub_source_path", fallback="av_stream/ch1").lstrip("/")
SUB_WIDTH = cfg.getint("streams", "sub_width", fallback=640)
SUB_HEIGHT = cfg.getint("streams", "sub_height", fallback=360)

SNAP_ENABLED = cfg.getboolean("snapshot", "enabled", fallback=True)
SNAP_INTERVAL = cfg.getint("snapshot", "refresh_interval", fallback=30)
SNAP_PATH = cfg.get("snapshot", "cache_path", fallback="/tmp/sonoff_snap.jpg")
SNAP_LOCK = threading.Lock()


def write_mediamtx_config():
    auth = f"{CAM_USER}:{CAM_PASS}@" if CAM_USER else ""
    yml = f"""# Auto-generated from {CONFIG_PATH} - do not edit manually
paths:
  {MAIN_PATH}:
    source: rtsp://{auth}{CAM_IP}:{CAM_RTSP_PORT}/{MAIN_SOURCE_PATH}
    rtspTransport: tcp
    sourceOnDemand: no
  {SUB_PATH}:
    source: rtsp://{auth}{CAM_IP}:{CAM_RTSP_PORT}/{SUB_SOURCE_PATH}
    rtspTransport: tcp
    sourceOnDemand: no
"""
    with open(MEDIAMTX_YML_PATH, "w") as f:
        f.write(yml)
    os.chmod(MEDIAMTX_YML_PATH, 0o600)


MAIN_RTSP_URI = f"rtsp://{HOST_IP}:{RTSP_PORT}/{MAIN_PATH}"
SUB_RTSP_URI = f"rtsp://{HOST_IP}:{RTSP_PORT}/{SUB_PATH}"
LOCAL_SUB_RTSP_URI = f"rtsp://127.0.0.1:{RTSP_PORT}/{SUB_PATH}"
SNAPSHOT_HTTP_URI = f"http://{HOST_IP}:{ONVIF_PORT}/snapshot.jpg"

ONVIF_CAPABILITIES = f"""<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope" xmlns:tds="http://www.onvif.org/ver10/device/wsdl" xmlns:trt="http://www.onvif.org/ver10/media/wsdl" xmlns:tt="http://www.onvif.org/ver10/schema">
  <SOAP-ENV:Body>
    <tds:GetCapabilitiesResponse><tds:Capabilities>
      <tds:Device><tds:XAddr>http://{HOST_IP}:{ONVIF_PORT}/onvif/device_service</tds:XAddr></tds:Device>
      <tds:Media>
        <tds:XAddr>http://{HOST_IP}:{ONVIF_PORT}/onvif/media_service</tds:XAddr>
        <tds:StreamingCapabilities>
          <tt:RTPMulticast>false</tt:RTPMulticast>
          <tt:RTP_TCP>true</tt:RTP_TCP>
          <tt:RTP_RTSP_TCP>true</tt:RTP_RTSP_TCP>
        </tds:StreamingCapabilities>
      </tds:Media>
    </tds:Capabilities></tds:GetCapabilitiesResponse>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>"""

ONVIF_MEDIA_SERVICE_CAPS = f"""<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope" xmlns:trt="http://www.onvif.org/ver10/media/wsdl">
  <SOAP-ENV:Body>
    <trt:GetServiceCapabilitiesResponse>
      <trt:Capabilities SnapshotUri="{str(SNAP_ENABLED).lower()}" Rotation="false" VideoSourceMode="false" OSD="false" TemporaryOSDText="false" EXICompression="false"/>
    </trt:GetServiceCapabilitiesResponse>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>"""

ONVIF_INFO = f"""<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope" xmlns:tds="http://www.onvif.org/ver10/device/wsdl">
  <SOAP-ENV:Body><tds:GetDeviceInformationResponse>
    <tds:Manufacturer>{MANUFACTURER}</tds:Manufacturer><tds:Model>{MODEL}</tds:Model>
    <tds:FirmwareVersion>{FIRMWARE}</tds:FirmwareVersion><tds:SerialNumber>{SERIAL}</tds:SerialNumber><tds:HardwareId>{HARDWARE_ID}</tds:HardwareId>
  </tds:GetDeviceInformationResponse></SOAP-ENV:Body>
</SOAP-ENV:Envelope>"""

ONVIF_PROFILES = f"""<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope" xmlns:trt="http://www.onvif.org/ver10/media/wsdl" xmlns:tt="http://www.onvif.org/ver10/schema">
  <SOAP-ENV:Body><trt:GetProfilesResponse>
    <trt:Profiles token="main" fixed="true"><tt:Name>MainStream</tt:Name>
      <tt:VideoEncoderConfiguration token="vec_main"><tt:Name>H264_Main</tt:Name><tt:Encoding>H264</tt:Encoding>
        <tt:Resolution><tt:Width>{MAIN_WIDTH}</tt:Width><tt:Height>{MAIN_HEIGHT}</tt:Height></tt:Resolution>
      </tt:VideoEncoderConfiguration>
    </trt:Profiles>
    <trt:Profiles token="sub" fixed="true"><tt:Name>SubStream</tt:Name>
      <tt:VideoEncoderConfiguration token="vec_sub"><tt:Name>H264_Sub</tt:Name><tt:Encoding>H264</tt:Encoding>
        <tt:Resolution><tt:Width>{SUB_WIDTH}</tt:Width><tt:Height>{SUB_HEIGHT}</tt:Height></tt:Resolution>
      </tt:VideoEncoderConfiguration>
    </trt:Profiles>
  </trt:GetProfilesResponse></SOAP-ENV:Body>
</SOAP-ENV:Envelope>"""

ONVIF_SNAPSHOT = f"""<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope" xmlns:trt="http://www.onvif.org/ver10/media/wsdl" xmlns:tt="http://www.onvif.org/ver10/schema">
  <SOAP-ENV:Body><trt:GetSnapshotUriResponse><trt:MediaUri>
    <tt:Uri>{SNAPSHOT_HTTP_URI}</tt:Uri>
    <tt:InvalidAfterConnect>false</tt:InvalidAfterConnect>
    <tt:InvalidAfterReboot>false</tt:InvalidAfterReboot>
    <tt:Timeout>PT10S</tt:Timeout>
  </trt:MediaUri></trt:GetSnapshotUriResponse></SOAP-ENV:Body>
</SOAP-ENV:Envelope>"""


def build_stream_uri(uri):
    return f"""<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope" xmlns:trt="http://www.onvif.org/ver10/media/wsdl" xmlns:tt="http://www.onvif.org/ver10/schema">
  <SOAP-ENV:Body><trt:GetStreamUriResponse><trt:MediaUri>
    <tt:Uri>{uri}</tt:Uri>
    <tt:InvalidAfterConnect>false</tt:InvalidAfterConnect>
    <tt:InvalidAfterReboot>false</tt:InvalidAfterReboot>
    <tt:Timeout>PT60S</tt:Timeout>
  </trt:MediaUri></trt:GetStreamUriResponse></SOAP-ENV:Body>
</SOAP-ENV:Envelope>"""


def refresh_snapshot(force=False):
    if not SNAP_ENABLED:
        return False
    with SNAP_LOCK:
        if not force and os.path.exists(SNAP_PATH) and (time.time() - os.path.getmtime(SNAP_PATH)) < max(1, SNAP_INTERVAL - 1):
            return True
        tmp = "/tmp/sonoff_snap.tmp.jpg"
        proc = subprocess.run(
            [
                "ffmpeg", "-nostdin", "-loglevel", "error", "-y",
                "-rtsp_transport", "tcp", "-i", LOCAL_SUB_RTSP_URI,
                "-frames:v", "1", "-f", "image2", "-q:v", "5", tmp,
            ],
            capture_output=True,
            timeout=6,
        )
        if proc.returncode == 0 and os.path.exists(tmp) and os.path.getsize(tmp) > 0:
            os.replace(tmp, SNAP_PATH)
            return True
        return os.path.exists(SNAP_PATH)


def bg_loop():
    while True:
        try:
            refresh_snapshot(force=True)
        except Exception:
            pass
        time.sleep(SNAP_INTERVAL)


class ReusableThreadingServer(socketserver.ThreadingMixIn, socketserver.TCPServer):
    allow_reuse_address = True
    daemon_threads = True


class Handler(http.server.BaseHTTPRequestHandler):
    def _serve_snapshot(self, send_body=True):
        if refresh_snapshot(force=False):
            with open(SNAP_PATH, "rb") as f:
                data = f.read()
            self.send_response(200)
            self.send_header("Content-Type", "image/jpeg")
            self.send_header("Content-Length", str(len(data)))
            self.end_headers()
            if send_body:
                self.wfile.write(data)
            return
        self.send_error(502)

    def do_HEAD(self):
        if self.path.startswith("/snapshot.jpg"):
            self._serve_snapshot(send_body=False)
        else:
            self.send_error(404)

    def do_GET(self):
        if self.path.startswith("/snapshot.jpg"):
            self._serve_snapshot(send_body=True)
        else:
            self.send_error(404)

    def do_POST(self):
        body = self.rfile.read(int(self.headers.get("Content-Length", 0))).decode(errors="ignore")
        if "GetDeviceInformation" in body:
            resp = ONVIF_INFO
        elif "GetServiceCapabilities" in body:
            resp = ONVIF_MEDIA_SERVICE_CAPS
        elif "GetProfiles" in body:
            resp = ONVIF_PROFILES
        elif "GetSnapshotUri" in body:
            resp = ONVIF_SNAPSHOT
        elif "GetStreamUri" in body:
            uri = SUB_RTSP_URI if ">sub<" in body else MAIN_RTSP_URI
            resp = build_stream_uri(uri)
        else:
            resp = ONVIF_CAPABILITIES
        self.send_response(200)
        self.send_header("Content-Type", "application/soap+xml; charset=utf-8")
        self.end_headers()
        self.wfile.write(resp.encode())

    def log_message(self, format, *args):
        pass


if __name__ == "__main__":
    if "--generate-mediamtx" in sys.argv:
        write_mediamtx_config()
        sys.exit(0)
    if SNAP_ENABLED:
        threading.Thread(target=bg_loop, daemon=True).start()
    ReusableThreadingServer(("0.0.0.0", ONVIF_PORT), Handler).serve_forever()
EOF
```

---

## 6. Systemd Services (`mediamtx.service` & `onvif-bridge.service`)

Configure `mediamtx.service` to generate `/opt/mediamtx.yml` via `ExecStartPre` before starting MediaMTX, and bind `onvif-bridge.service` with `Requires=` and `PartOf=` so restarting `mediamtx.service` after editing `/etc/onvif_bridge.conf` updates both services automatically:

```bash
cat << 'EOF' > /etc/systemd/system/mediamtx.service
[Unit]
Description=MediaMTX RTSP Proxy Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStartPre=/usr/bin/python3 /opt/onvif_bridge.py --generate-mediamtx
ExecStart=/opt/mediamtx /opt/mediamtx.yml
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

cat << 'EOF' > /etc/systemd/system/onvif-bridge.service
[Unit]
Description=Lightweight ONVIF-to-RTSP Bridge for Sonoff CAM-S3W
After=network-online.target mediamtx.service
Wants=network-online.target
Requires=mediamtx.service
PartOf=mediamtx.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/onvif_bridge.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now mediamtx.service onvif-bridge.service
```

---

## 7. Verification & Maintenance

1. **Verify generated config, services, ports, and snapshot endpoint:**

   ```bash
   cat /opt/mediamtx.yml
   systemctl status mediamtx.service onvif-bridge.service --no-pager
   ss -tlnp | grep -E '8000|8554'
   curl -sI http://127.0.0.1:8000/snapshot.jpg | head -n 4
   ```
   Confirm `/opt/mediamtx.yml` was auto-generated from `/etc/onvif_bridge.conf`, both services show `active (running)`, ports `8000` and `8554` are listening, and `/snapshot.jpg` returns `HTTP/1.0 200 OK` with `Content-Type: image/jpeg`.

3. **Adopt in UniFi Protect:**
   * Open **UniFi Protect > Devices > Add Device** (or **Advanced Adoption**).
   * Adopt **`10.15.30.242:8000`** with `username` (`sonoff`) and `password` (`camfeed`).
   * Verify both **HQ** (`2304x1296`) and **Low** (`640x360`) streams play cleanly.

4. **Future Configuration Changes:**
   * **Edited `/etc/onvif_bridge.conf`** (IPs, credentials, stream paths, snapshot interval):
     `
     systemctl restart mediamtx.service
     `
     *Regenerates `/opt/mediamtx.yml` via `ExecStartPre` and automatically restarts both `mediamtx.service` and `onvif-bridge.service` (via `PartOf=mediamtx.service`). No `daemon-reload` is needed.*
   * **Edited `/opt/onvif_bridge.py`** (Python bridge logic only):
     `
     systemctl restart onvif-bridge.service
     `
   * **Edited `/etc/systemd/system/*.service`** (systemd unit files):
     `
     systemctl daemon-reload && systemctl restart mediamtx.service
     `
---

## 8. References
[AI Port ONVIF troubleshooting](https://community.ui.com/questions/AI-Port-ONVIF-troubleshooting/4bf47d15-3f80-4b71-ae95-ad700b95fa2a): Packet and API traces revealed that UniFi Protect uses the Node.js agsh/onvif library. When Protect hits any SOAP/HTTP parsing error after authentication (400 Bad Request on /proxy/protect/api/third-party-cameras/adopt), the UI masks the error and displays "Invalid credentials" rather than the actual SOAP fault.

[Adding Third-Party Camera "Invalid Credentials" Due to SOAP Error Response](https://community.ui.com/questions/Adding-Third-Party-Camera-Invalid-Credentials-Due-to-SOAP-Error-Response/7e09bc87-6ce9-4a5d-ae69-90b7c902afeb): Packet captures on port 8000 confirmed that WS-Security password digest authentication succeeded, but Protect immediately failed with "Invalid Credentials" when querying Media2 / stream profiles that returned a SOAP InvalidArgVal fault—and users noted that disabling strict "ONVIF Authentication" (WS-Security enforcement) on the camera resolved adoption.

[Error in UniFi Protect adopting Amcrest camera — "Invalid credentials"](https://community.ui.com/questions/Error-in-Unifi-Protect-adopting-Amcrest-camera-greater-Invalid-credentials-Please-check-and-try-aga/c85fcbca-3c54-470f-ae6f-96747f9bb24b): Users repeatedly hit "Invalid credentials. Please check and try again." until toggling the camera's ONVIF Authentication setting off—bypassing the camera's non-standard WS-Security check while still authenticating the RTSP feed.

[Older Hikvision ONVIF — incorrect password error](https://community.ui.com/questions/041f7bf1-b5b1-43ca-9267-0de4343a952b): Inspecting /srv/unifi-protect/logs/cameras.thirdParty.log confirmed that Protect throws an incorrect-password style failure when getOnvifDefaultCameraSettings() fails to parse non-standard videoEncoderConfiguration / resolution fields in the camera's SOAP response.
