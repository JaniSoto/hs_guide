


<div class="phase-header">
  <span class="phase-num">12</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 12 · Local AI Setup</p>
    <h1>Secretary-Boss Architecture</h1>
    <p class="phase-header-sub">Deploy a dual-GPU, locally hosted AI setup. A hyper-fast "Secretary" model runs instantly on the iGPU, automatically waking the eGPU "Boss" model for heavy compute tasks.</p>
  </div>
</div>

---

## Step 1 — Hardware & OS Pre-flight

We must configure the kernel and Docker to handle the dual-GPU workload without freezing. 

**1. BIOS Configuration (Mandatory)**
Reboot into the BIOS and ensure your **UMA Frame Buffer** (iGPU VRAM allocation) is explicitly set to **16GB**. Leaving it on "Auto" will result in crashes.

**2. Apply Kernel Parameters**
Run these to stabilize ROCm (AMD's compute engine) for the 780M iGPU:

```bash
sudo rpm-ostree kargs \
  --append-if-missing=amdgpu.sg_display=0 \
  --append-if-missing=amdgpu.noretry=0 \
  --append-if-missing=amdgpu.cwsr_enable=0
```

**3. Get your Linux Group IDs**
Your Docker containers need exact hardware permissions to talk to the AMD GPUs. Run these two commands and **write down the numbers** (e.g., 44 and 107):

```bash
getent group video | cut -d: -f3
getent group render | cut -d: -f3
```

!!! info "Reboot Required"
    You must reboot your machine now for the kernel parameters to apply. Run `systemctl reboot` and reconnect via SSH.

---

## Step 2 — Deploy the Dual-GPU Stack

We will deploy **two** separate Ollama containers — one strictly bound to the iGPU (`renderD128`) and one bound to the eGPU (`renderD129`). Open WebUI will connect to both.

!!! warning "Crucial: Power on the eGPU first!"
    Before running the Docker Compose command, you MUST turn on the eGPU using the script from Phase 11:
    `sudo gpu-on.sh`
    Docker needs to "see" `/dev/dri/renderD129` during the initial container creation.

Create a directory for your stack and open a new compose file:

```bash
mkdir -p ~/docker/ai-stack && cd ~/docker/ai-stack
nano docker-compose.yml
```

Paste the following configuration. **Replace `YOUR_VIDEO_GID` and `YOUR_RENDER_GID`** with your numbers.

```yaml
services:
  # ── THE SECRETARY (iGPU - Always On) ────────────────────────
  ollama-igpu:
    image: ollama/ollama:rocm
    container_name: ollama-igpu
    restart: unless-stopped
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri/renderD128:/dev/dri/renderD128  # Locked to 780M
    group_add:
      - YOUR_VIDEO_GID
      - YOUR_RENDER_GID
    environment:
      - HSA_OVERRIDE_GFX_VERSION=11.0.0  # Spoof 780M for ROCm compatibility
      - OLLAMA_FLASH_ATTENTION=false
    volumes:
      - ./ollama_igpu_data:/root/.ollama:z
    networks:
      - nextcloud-aio

  # ── THE BOSS (eGPU - On Demand) ─────────────────────────────
  ollama-egpu:
    image: ollama/ollama:rocm
    container_name: ollama-egpu
    restart: "no"  # Do not auto-restart; gpu-on.sh manages this
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri/renderD129:/dev/dri/renderD129  # Locked to eGPU
    group_add:
      - YOUR_VIDEO_GID
      - YOUR_RENDER_GID
    volumes:
      - ./ollama_egpu_data:/root/.ollama:z
    networks:
      - nextcloud-aio

  # ── OPEN WEBUI (The Interface) ──────────────────────────────
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      # Connects to both Ollama instances
      - OLLAMA_BASE_URLS=http://ollama-igpu:11434;http://ollama-egpu:11434
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

## Step 3 — Download the Models

We will use a fast 3B model for the Secretary and a highly capable 14B model for the Boss.

**1. Pull the Secretary Model (iGPU)**
```bash
docker exec ollama-igpu ollama pull qwen2.5:3b
```

**2. Pull the Boss Model (eGPU)**
```bash
docker exec ollama-egpu ollama pull qwen2.5:14b-instruct-q4_K_M
```

---

## Step 4 — Open WebUI Optimization

Open your browser and navigate to `http://<your-server-ip>:3000`. Create your admin account.
Go to **Settings** (Admin Panel) → **Workspace** → **Settings**. We must prevent Open WebUI from bloating VRAM with secondary models.

*   Under **Capabilities**, **uncheck** Vision, File Upload, Web Search, Image Generation, Code Interpreter, and Citations. Leave **Status Updates** and **Builtin Tools** checked.
*   Under **Builtin Tools**, **uncheck** Memory, Chat History, Web Search, Image Generation, and Code Interpreter. Leave **Time & Calculation** checked.
*   Click **Save**.

---

## Step 5 — Native Python Tools

We will leverage Open WebUI's native Python engine. The "Secretary" will have access to these to manage your home, files, and intelligently wake the eGPU when needed.

Go to **Workspace** (left sidebar) → **Tools** → **Create Tool**. You will create three separate tools using the code blocks below.

=== "Wake Boss AI Tool"

    **Name:** `wake_boss`
    **Description:** *Triggers the eGPU power cycle.*
    **Update:** Replace `192.168.1.xxx:PORT` with your webhook server IP and port from Phase 11.

    ```python
    """
    title: Wake Boss AI
    description: Powers on the eGPU to enable the heavy-duty "Boss" model for complex tasks.
    author: LocalAdmin
    version: 1.0
    """
    import requests

    class Tools:
        def __init__(self):
            # UPDATE THIS LINE
            self.webhook_url = "http://192.168.1.xxx:PORT/gpu-on"

        def wake_boss_ai(self) -> str:
            """
            USE THIS IMMEDIATELY if the user asks a highly complex question, requests heavy coding, 
            needs deep reasoning, or explicitly asks for "the boss", "more power", or "the big AI".
            """
            try:
                requests.get(self.webhook_url, timeout=5)
                return "Action Successful. Tell the user EXACTLY this: 'Hold on, let me enable more computing power. The eGPU is waking up. Please switch to the Boss model in the top left menu in about 30 seconds.'"
            except Exception as e:
                return f"Failed to wake the eGPU: {e}"
    ```

=== "Nextcloud Memory Tool"

    **Name:** `nextcloud_memory`
    **Description:** *Read, write, and remember lists in Nextcloud.*
    **Prerequisite:** Create an `AI_Memory` folder in Nextcloud and generate an App Password in your Nextcloud Security settings.

    ```python
    """
    title: Nextcloud Memory Manager
    description: Read, write, and remember lists or notes natively in Nextcloud via WebDAV.
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
            if not filename.endswith('.txt'): filename += '.txt'
            file_url = f"{self.base_url}/{urllib.parse.quote(filename)}"
            current_content = ""
            
            if mode == "append":
                try:
                    res = requests.get(file_url, auth=(self.username, self.app_password))
                    if res.status_code == 200: current_content = res.text + "\n"
                except Exception: pass
            
            new_content = current_content + content
            response = requests.put(file_url, auth=(self.username, self.app_password), data=new_content.encode('utf-8'))
            
            if response.status_code in[201, 204]: return f"Success: Saved to {filename}."
            return f"Error saving file: {response.status_code} - {response.text}"

        def read_from_nextcloud(self, filename: str) -> str:
            """
            Reads the contents of a list or memory file. Use this BEFORE answering what is on a list.
            :param filename: The name of the file (e.g., "grocery_list.txt")
            """
            if not filename.endswith('.txt'): filename += '.txt'
            file_url = f"{self.base_url}/{urllib.parse.quote(filename)}"
            response = requests.get(file_url, auth=(self.username, self.app_password))
            
            if response.status_code == 200: return f"Contents of {filename}:\n{response.text}"
            return f"The file {filename} is empty or does not exist."
    ```

=== "Home Assistant Tool"

    **Name:** `smart_home_controller`
    **Description:** *Control smart home devices via REST API.*
    **Prerequisite:** Generate a Long-Lived Access Token in Home Assistant.

    ```python
    """
    title: Smart Home Controller
    description: Control and check physical smart home devices like lights, blinds, and AC.
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
            Controls a smart home device.
            :param entity_id: The exact ID of the device (e.g., "light.living_room", "climate.ac")
            :param action: "turn_on", "turn_off", "open_cover", "close_cover", or "toggle".
            """
            domain = entity_id.split('.')[0]
            url = f"{self.ha_url}/services/{domain}/{action}"
            
            response = requests.post(url, headers=self.headers, json={"entity_id": entity_id})
            if response.status_code == 200: return f"Successfully triggered {action} on {entity_id}."
            return f"Failed to control device: {response.text}"

        def get_device_state(self, entity_id: str) -> str:
            """
            Checks the current status of a device. ALWAYS use this before assuming a device's state.
            :param entity_id: The exact ID of the device (e.g., "light.living_room")
            """
            url = f"{self.ha_url}/states/{entity_id}"
            response = requests.get(url, headers=self.headers)
            
            if response.status_code == 200:
                state_data = response.json()
                state = state_data.get('state', 'unknown')
                return f"The current state of {entity_id} is '{state}'."
            return f"Could not find device {entity_id}."
    ```

---

## Step 6 — System Prompts & Model Setup

We will create two separate System Personas in Open WebUI. Go to **Workspace** → **Models** and click the **+ (Create Model)** button for each.

=== "1. The Secretary (iGPU)"

    *   **Name:** `Secretary`
    *   **Base Model:** `qwen2.5:3b`
    *   **System Prompt:**
        ```text
        You are the Secretary, a hyper-fast, localized Smart Home and Memory Assistant.
        You handle simple tasks immediately. If a task is too complex, you must call the Boss.

        CORE DIRECTIVES:
        1. HOME CONTROL: If asked to turn on/off devices, control AC, or open blinds, IMMEDIATELY use the Smart Home Controller tool.
        2. MEMORY & LISTS: If asked to "remember" something or check a list, IMMEDIATELY use the Nextcloud Memory Manager tool. Filenames must always end in .txt and be simple (e.g., "groceries.txt").
        3. ESCALATION (WAKE BOSS): If the user asks a highly complex question, asks for code, deep reasoning, or says "I need the boss", IMMEDIATELY use the Wake Boss AI tool.
        4. NEVER HALLUCINATE: Never guess the state of a home device or the contents of a list. Use your tools to check first.
        5. BE ULTRA-CONCISE: Once a tool executes successfully, confirm the action in ONE short sentence. Never explain how you did it.
        ```
    *   **Advanced Params (Click Show):** 
        *   `num_ctx`: `4096`
        *   `num_batch`: `256`
        *   `Temperature`: `0.1`

    *Click **Save & Update**.*

=== "2. The Boss (eGPU)"

    *   **Name:** `Boss`
    *   **Base Model:** `qwen2.5:14b-instruct-q4_K_M`
    *   **System Prompt:**
        ```text
        You are Jarvis, the Boss AI. You have been awakened because the user requires heavy computational reasoning, coding, or complex analysis. 
        Provide highly detailed, intelligent, and accurate responses. You do not need to control the smart home; focus strictly on the complex task the user gives you.
        ```
    *   **Advanced Params (Click Show):** 
        *   `num_ctx`: `8192`
        *   `use_mmap`: `Enabled`
        *   `num_batch`: `512`
        *   `Temperature`: `0.3`

    *Click **Save & Update**.*

---

## Step 7 — Activate the Workflow

1. Go back to your main Chat window.
2. Select the **Secretary** model from the top-left dropdown.
3. Click the **+** (Tools) icon next to the chat input and toggle **ON** all three tools: `wake_boss`, `nextcloud_memory`, and `smart_home_controller`.
4. Test the pipeline by saying: *"I need you to write a complex Python script for me."*

**Expected Result:**
The Secretary will instantly execute the `wake_boss` tool, respond with *"Hold on, let me enable more computing power..."*, and behind the scenes, your eGPU will physically power up. In about 30 seconds, you can switch the top-left dropdown to **Boss** and continue your complex task. 

When you are done, the 30-minute idle timer from Phase 11 will automatically power the eGPU back down, seamlessly leaving the Secretary running on your iGPU.

---

## ✅ Phase 12 Complete

You have successfully established a highly reliable, hardware-separated dual-GPU AI pipeline. Routine queries run at zero-latency on 35W of power, while heavy reasoning scales to 290W entirely on demand.
