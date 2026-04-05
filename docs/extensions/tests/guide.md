


<div class="phase-header">
  <span class="phase-num">12</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 12 · Local AI Setup</p>
    <h1>Jarvis Native Architecture</h1>
    <p class="phase-header-sub">Deploy a locally hosted, privacy-first AI Assistant optimized for the Ryzen 780M with native Nextcloud and Home Assistant integrations.</p>
  </div>
</div>

---

## Step 1 — Hardware & OS Pre-flight

To run a 14-Billion parameter model locally with tool-calling capabilities, we must explicitly configure your AMD APU and kernel to handle the massive VRAM workload without freezing the system.

**1. BIOS Configuration (Mandatory)**
Reboot your machine into the BIOS and ensure your **UMA Frame Buffer** (VRAM allocation) is explicitly set to **16GB**. Leaving it on "Auto" will result in out-of-memory crashes.

**2. Apply Bazzite Kernel Parameters**
Run these commands to stabilize ROCm (AMD's compute engine) and prevent the system from hanging during heavy text generation:

```bash
sudo rpm-ostree kargs --append=amdgpu.sg_display=0
sudo rpm-ostree kargs --append=amdgpu.noretry=0
sudo rpm-ostree kargs --append=amdgpu.cwsr_enable=0
```

**3. Get your Linux Group IDs**
Your Docker container needs exact hardware permissions to talk to the Radeon 780M. Run these two commands and **write down the numbers** (e.g., 44 and 107):

```bash
getent group video | cut -d: -f3
getent group render | cut -d: -f3
```

!!! info "Reboot Required"
    You must reboot your machine now for the kernel parameters to apply. Run `systemctl reboot` and reconnect via SSH.

---

## Step 2 — Deploy the AI Stack

We will deploy Ollama (the LLM engine) and Open WebUI (the chat interface). They will join the `nextcloud-aio` network you created in Phase 5 so they can communicate securely with your existing setup.

Create a directory for your stack and open a new compose file:

```bash
mkdir -p ~/docker/ai-stack && cd ~/docker/ai-stack
nano docker-compose.yml
```

Paste the following configuration. **Make sure to replace `YOUR_VIDEO_GID` and `YOUR_RENDER_GID`** with the numbers you wrote down in Step 1.

```yaml
services:
  ollama:
    image: ollama/ollama:rocm
    container_name: ollama
    restart: unless-stopped
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    group_add:
      - YOUR_VIDEO_GID  # Replace with video GID
      - YOUR_RENDER_GID # Replace with render GID
    environment:
      # Spoofing the 780M (gfx1103) as gfx1100 to ensure ROCm compatibility
      - HSA_OVERRIDE_GFX_VERSION=11.0.0
      - OLLAMA_FLASH_ATTENTION=false
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_MAX_LOADED_MODELS=1
    volumes:
      - ./ollama_data:/root/.ollama:z
    networks:
      - nextcloud-aio

  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - ./webui_data:/app/backend/data:z
    networks:
      - nextcloud-aio

networks:
  nextcloud-aio:
    external: true
```

Start the stack:

```bash
docker compose up -d
```

---

## Step 3 — The VRAM Diet & Model Configuration

The Qwen 2.5 14B model is incredibly smart but requires ~9.3GB of VRAM just to load. If we do not restrict its "context window" (memory size), it will demand over 16GB, causing the AMD driver to Segmentation Fault (`SIGSEGV`) and crash the runner.

**1. Pull the Model**
Download the model directly into the Ollama container:
```bash
docker exec ollama ollama pull qwen2.5:14b-instruct-q4_K_M
```

**2. Strip Open WebUI Bloat**
Open your browser and navigate to `http://<your-server-ip>:3000`. Create your admin account.
Go to **Settings** (Admin Panel) → **Workspace** → **Settings**. We must prevent Open WebUI from loading secondary embedding models into VRAM.

*   Under **Capabilities**, **uncheck** Vision, File Upload, File Context, Web Search, Image Generation, Code Interpreter, Usage, and Citations. Leave **Status Updates** and **Builtin Tools** checked.
*   Under **Builtin Tools**, **uncheck** Memory, Chat History, Notes, Knowledge Base, Channels, Web Search, Image Generation, and Code Interpreter. Leave **Time & Calculation** checked.
*   Click **Save**.

**3. Create the Jarvis Persona & Apply the VRAM Limits**
Go to **Workspace** (left sidebar) → **Models** and click the **+ (Create Model)** button.

*   **Name:** `Jarvis`
*   **Base Model:** `qwen2.5:14b-instruct-q4_K_M`
*   **System Prompt:** Paste the following strict logic exactly as written:

```text
You are Jarvis, a highly intelligent, localized Smart Home and Memory Assistant.
You have access to tools. Use them immediately when requested.

CORE DIRECTIVES:
1. HOME CONTROL: If asked to turn on/off devices, control AC, or open blinds, IMMEDIATELY use the Smart Home Controller tool.
2. MEMORY & LISTS: If asked to "remember" something, "make a list", or check a list, IMMEDIATELY use the Nextcloud Memory Manager tool. File names must always end in .txt.
3. STRICT NAMING CONVENTION: When creating or looking for files, ALWAYS use ultra-simple, lowercase, one-or-two word filenames (e.g., "hana_gifts.txt", "groceries.txt", "todo.txt"). Never use words like "list" or "ideas" in the filename unless explicitly requested.
4. NEVER HALLUCINATE: Never guess the state of a home device or the contents of a list. ALWAYS use your tools to check first.
5. SELF-CORRECTION: If you try to read a list and the tool says it is empty or does not exist, silently guess the next most likely simple filename and check again before telling the user it doesn't exist.
6. BE ULTRA-CONCISE: Once a tool executes successfully, confirm the action in ONE short sentence. Never explain how you did it.
```

!!! danger "Critical VRAM Settings"
    Scroll down to **Advanced Params** and click **Show**. You MUST set **`num_ctx`** to `8192`. The default is `32768`, which *will* instantly crash your GPU. 
    Also set **`use_mmap`** to enabled, **`num_batch`** to `512`, and **`Temperature`** to `0.1`.

Click **Save & Update**.

---

## Step 4 — Native Python Tools

We will leverage Open WebUI's native Python tool engine to execute scripts directly, completely avoiding Docker volume permission issues.

### Tool 1: Nextcloud WebDAV Memory

This tool allows Jarvis to read and write lists seamlessly into your Nextcloud setup from Phase 10. 

**Prerequisite:** 
1. Log into Nextcloud (`https://yourname.duckdns.org`).
2. Create a folder named `AI_Memory` in your main files area.
3. Click your Profile Icon → **Personal Settings** → **Security**.
4. Scroll to the *very bottom* to **Devices & sessions**. Type "Jarvis", click **Create new app password**, and copy the password provided.

Go to Open WebUI → **Workspace** → **Tools** → **Create Tool**.
Name it `nextcloud_memory` and paste this code. **Update lines 13, 14, and 15**:

```python
"""
title: Nextcloud Memory Manager
description: Read, write, and remember lists, passwords, and notes natively in Nextcloud via WebDAV.
author: LocalAdmin
version: 1.3
"""
import requests
import urllib.parse

class Tools:
    def __init__(self):
        # UPDATE THESE 3 LINES
        self.nc_url = "https://yourname.duckdns.org" # Your Phase 10 Domain
        self.username = "your_username"
        self.app_password = "your_app_password"
        
        self.folder_path = "AI_Memory" 
        self.base_url = f"{self.nc_url}/remote.php/dav/files/{self.username}/{self.folder_path}"

    def write_to_nextcloud(self, filename: str, content: str, mode: str = "append") -> str:
        """
        Writes or appends to a memory file or list (e.g., "grocery_list.txt").
        :param filename: The name of the file. MUST end in .txt
        :param content: The text to save.
        :param mode: "append" to add to an existing list, or "overwrite" to replace.
        """
        if not filename.endswith('.txt'): 
            filename += '.txt'
            
        file_url = f"{self.base_url}/{urllib.parse.quote(filename)}"
        current_content = ""
        
        if mode == "append":
            try:
                res = requests.get(file_url, auth=(self.username, self.app_password))
                if res.status_code == 200: 
                    current_content = res.text + "\n"
            except Exception: 
                pass
        
        new_content = current_content + content
        response = requests.put(file_url, auth=(self.username, self.app_password), data=new_content.encode('utf-8'))
        
        if response.status_code in[201, 204]:
            return f"Success: Saved to {filename}."
        return f"Error saving file: {response.status_code} - {response.text}"

    def read_from_nextcloud(self, filename: str) -> str:
        """
        Reads the contents of a list or memory file. Use this BEFORE answering what is on a list.
        :param filename: The name of the file (e.g., "grocery_list.txt")
        """
        if not filename.endswith('.txt'): 
            filename += '.txt'
            
        file_url = f"{self.base_url}/{urllib.parse.quote(filename)}"
        response = requests.get(file_url, auth=(self.username, self.app_password))
        
        if response.status_code == 200: 
            return f"Contents of {filename}:\n{response.text}"
        return f"The file {filename} is empty or does not exist."
```
Click **Save**.

### Tool 2: Home Assistant Controller

**Prerequisite:** In Home Assistant, click your Profile picture (bottom left) → Security → **Create Long-Lived Access Token**.

Create another tool, name it `smart_home_controller`. **Update lines 13 & 14**:

```python
"""
title: Smart Home Controller
description: Control and check physical smart home devices like lights, blinds, and AC via REST API.
author: LocalAdmin
version: 1.1
"""
import requests

class Tools:
    def __init__(self):
        # UPDATE THESE 2 LINES
        self.ha_url = "http://192.168.1.xxx:8123/api" # Your HA LAN IP
        self.ha_token = "YOUR_LONG_LIVED_ACCESS_TOKEN"
        
        self.headers = {
            "Authorization": f"Bearer {self.ha_token}", 
            "Content-Type": "application/json"
        }

    def control_device(self, entity_id: str, action: str) -> str:
        """
        Controls a smart home device (turn on/off, open/close, toggle).
        :param entity_id: The exact ID of the device (e.g., "light.living_room", "cover.blinds", "climate.ac")
        :param action: The action to perform: "turn_on", "turn_off", "open_cover", "close_cover", or "toggle".
        """
        domain = entity_id.split('.')[0]
        url = f"{self.ha_url}/services/{domain}/{action}"
        
        response = requests.post(url, headers=self.headers, json={"entity_id": entity_id})
        if response.status_code == 200:
            return f"Successfully triggered {action} on {entity_id}."
        return f"Failed to control device: {response.text}"

    def get_device_state(self, entity_id: str) -> str:
        """
        Checks the current status of a smart home device. ALWAYS use this before assuming a device's state.
        :param entity_id: The exact ID of the device (e.g., "light.living_room")
        """
        url = f"{self.ha_url}/states/{entity_id}"
        response = requests.get(url, headers=self.headers)
        
        if response.status_code == 200:
            state_data = response.json()
            state = state_data.get('state', 'unknown')
            return f"The current state of {entity_id} is '{state}'."
        return f"Could not find device {entity_id}. Check if the entity_id is correct."
```
Click **Save**.

---

## Step 5 — Activate and Use

1. Go back to your main Chat window.
2. Select your **Jarvis** model from the top-left dropdown.
3. Click the **+** (Tools) icon next to the chat input and toggle **ON** both `nextcloud_memory` and `smart_home_controller`.
4. Test the pipeline by asking: *"Create a list of groceries and add milk and bread."*

!!! info "Mobile Voice Integration"
    Open WebUI includes native Speech-to-Text. Navigate to `http://<your-server-ip>:3000` on your smartphone browser, tap your browser's menu, and select **"Add to Home Screen"**. You now have a native app with a microphone button to speak directly to Jarvis, who will automatically use your tools based on conversational intent.

---

## ✅ Phase 12 Complete

Your Ryzen 780M is now natively executing a 14B parameter AI, routed directly to your personal cloud and smart home hardware, with perfectly tuned VRAM management.
