 # Step-by-Step Guide: Automated Server Update Notifications with n8n and Home Assistant

This guide walks you through creating an automation that checks a Debian/Ubuntu server for APT updates, sends a push notification to your phone via Home Assistant with an approve/reject option, and installs updates if approved. The solution uses n8n for workflow orchestration and Home Assistant for notifications.

Here is the step-by-step guide in Markdown format. You can copy the text below into a `.md` file (like `README.md`) or paste it into documentation tools like Notion or Obsidian.


## 🛠️ Prerequisites

Before starting, ensure you have these essential components configured:

1.  **SSH Key Credential:** You must have an SSH Key pair set up. The user must have `NOPASSWD` configured in the `/etc/sudoers` file on the server for `sudo apt update` and `sudo apt upgrade -y` to prevent password prompts.
2.  **Home Assistant Credential:** An **HTTP Bearer Auth** credential containing your Home Assistant **Long-Lived Access Token**.
3.  **Network Access:** The n8n server must be able to reach your Home Assistant instance (e.g., `http://10.0.30.2:8123`) and your Raspberry Pi/Ubuntu server.

-----

## 🚀 Step 1: Create the Update Check Branch

This section sets up the scheduled check, SSH command, and data parsing.

### 1\. Schedule Trigger Node

  * **Node Name:** `Schedule Trigger`
  * **Function:** Initiates the check at regular intervals.
  * **Settings:**
      * **Rule:** Interval
      * **Interval:** `1` Hours (or your desired frequency)

### 2\. Check for Updates (SSH Node)

  * **Node Name:** `Check for Updates`
  * **Function:** Connects to the server and lists upgradable packages.
  * **Settings:**
      * **Authentication:** Private Key (Select your "Nebula Private Key")
      * **Command:**
        ```bash
        sudo apt update && sudo apt list --upgradable
        ```

### 3\. Phrase Updates (Code Node)

  * **Node Name:** `Phrase Updates`
  * **Function:** Parses the messy SSH output into a clean list of packages.
  * **Settings:**
      * **Language:** JavaScript
      * **Mode:** Run Once for Each Item
      * **Code:**
        ```javascript
        const stdout = item.json.stdout;  // SSH output is in 'stdout' field

        if (!stdout || typeof stdout !== 'string') {
          return { json: { updates: '', rawOutput: '' } };  // No valid output
        }

        // Filter for lines that contain update info
        const lines = stdout.split('\n').filter(line => line.includes('[upgradable from:'));

        if (lines.length === 0) {
          return { json: { updates: '', rawOutput: stdout } };  // No updates
        }

        // Format the list
        const updateList = lines.map(line => {
          const parts = line.trim().split(' ');
          if (parts.length > 3) {
            const packageName = parts[0].split('/')[0];  // Extract package name
            const newVersion = parts[1];  // New version
            const oldVersionMatch = line.match(/\[upgradable from: (.*?)\]/);
            const oldVersion = oldVersionMatch ? oldVersionMatch[1] : 'unknown';  // Old version
            return `${packageName} (${oldVersion} -> ${newVersion})`;
          }
          return '';
        }).filter(pkg => pkg);  // Remove empty entries

        const updates = updateList.join('\n');

        return { json: { updates: updates, rawOutput: stdout } };
        ```

### 4\. If Node

  * **Node Name:** `If`
  * **Function:** Checks if the `updates` field is empty.
  * **Settings:**
      * **Value 1:** `={{ $json.updates }}`
      * **Operation:** Not Empty

### 5\. Notification Node (HTTP Request)

  * **Node Name:** `Notification Node`
  * **Function:** Sends the interactive notification to Home Assistant if updates exist.
  * **Settings:**
      * **Method:** POST
      * **URL:** `http://<YOUR_HA_IP>:8123/api/services/notify/mobile_app_<YOUR_DEVICE_ID>`
      * **Authentication:** Generic Credential Type -\> HTTP Bearer Auth (Select your HA credential)
      * **Headers:**
          * `content-type`: `application/json`
      * **Body:** JSON
      * **JSON/Expression:**
        ```json
        {{
          JSON.stringify({
            title: "Nebula Updates",
            message: $json.updates || "No updates available",
            data: {
              actions: [
                { 
                  action: "APPROVE_UPDATES", 
                  title: "Install" 
                },
                { 
                  action: "DISMISS", 
                  title: "Dismiss" 
                }
              ]
            }
          })
        }}
        ```

-----

## 🏗️ Step 2: Create the Install Updates Branch

This branch handles the actual installation when you press the "Install" button on your phone.

### 6\. Webhook Node

  * **Node Name:** `Webhook`
  * **Function:** Receives the callback from Home Assistant when the button is pressed.
  * **Settings:**
      * **HTTP Method:** POST
      * **Path:** `/approve-updates`
      * **Note:** You must configure your Home Assistant automation to call this Webhook URL when the `APPROVE_UPDATES` event is fired.

### 7\. Apply Updates (SSH Node)

  * **Node Name:** `Apply updates`
  * **Function:** Performs the upgrade on the server.
  * **Settings:**
      * **Authentication:** Private Key (Select your "Nebula Private Key")
      * **Command:**
        ```bash
        sudo apt upgrade -y && sudo apt autoremove
        ```
      * **Important:** The `-y` flag is required to bypass the "Are you sure?" prompt.

### 8\. Notification Node 2 (HTTP Request)

  * **Node Name:** `Notification Node 2`
  * **Function:** Confirms the installation was successful.
  * **Settings:**
      * **Method:** POST
      * **URL:** `http://<YOUR_HA_IP>:8123/api/services/notify/mobile_app_<YOUR_DEVICE_ID>`
      * **Headers:** `content-type`: `application/json`
      * **Body:** JSON
      * **JSON/Expression:**
        ```json
        {{ JSON.stringify({
          title: "Nebula Updates Installed",
          message: $json.updates || "Updates have successfully been installed"
        }) }}
        ```

-----

## 🔗 Step 3: Connections

Connect the nodes in this order:

1.  `Schedule Trigger` → `Check for Updates`
2.  `Check for Updates` → `Phrase Updates`
3.  `Phrase Updates` → `If`
4.  `If` (True) → `Notification Node`
5.  `Webhook` → `Apply updates`
6.  `Apply updates` → `Notification Node 2`
