 # Step-by-Step Guide: Automated Server Update Notifications with n8n and Home Assistant

This guide walks you through creating an automation that checks a Debian/Ubuntu server for APT updates, sends a push notification to your phone via Home Assistant with an approve/reject option, and installs updates if approved. The solution uses n8n for workflow orchestration and Home Assistant for notifications.

## Prerequisites
- **n8n**: Installed and running (self-hosted or cloud; version 1.x recommended as of July 2025).
- **Home Assistant**: Running with the Mobile App integration enabled on your phone.
- **Server**: A Debian/Ubuntu-based server with SSH access and `sudo` privileges for `apt` commands.
- **Tools**: Text editor, terminal, and HA companion app on your phone.

## Step 1: Set Up the Check and Notify Workflow
This workflow runs on a schedule, checks for updates, and sends a notification if any are available.

### 1.1 Create the Workflow
- In n8n, click **Add Workflow** and name it (e.g., `Server_Update_Check`).
- Add nodes as described below.

### 1.2 Add a Schedule Trigger
- **Node**: Schedule
- **Settings**:
  - Mode: "Every Day" or Cron: `0 8 * * *` (runs at 8 AM daily).
- This starts the workflow on a schedule.

### 1.3 Add an SSH Node to Check Updates
- **Node**: SSH
- **Settings**:
  - Credentials: Create new SSH credentials:
    - Host: Your server IP (e.g., `10.0.30.2`).
    - Port: 22 (default).
    - Username: Your SSH user (e.g., `user`).
    - Private Key: Paste your SSH private key (or use password if no key).
  - Command: `sudo apt update && sudo apt list --upgradable`
- Connect to the Schedule node.

### 1.4 Add a Code Node to Parse Updates
- **Node**: Code
- **Settings**:
  - Mode: "Run Once for Each Item"
  - JavaScript Code:
    ```javascript
    const stdout = item.json.stdout;  // SSH output is in 'stdout' field

    if (!stdout || typeof stdout !== 'string') {
      return { json: { updates: '', rawOutput: '' } };  // No valid output
    }

    const lines = stdout.split('\n').filter(line => line.includes('[upgradable from:'));  // Keep update lines

    if (lines.length === 0) {
      return { json: { updates: '', rawOutput: stdout } };  // No updates
    }

    const updateList = lines.map(line => {
      const parts = line.trim().split(' ');
      if (parts.length > 0) {
        return parts[0].split('/')[0];  // Extract package name
      }
      return '';
    }).filter(pkg => pkg);  // Remove empty entries

    return { json: { updates: updateList.join('\n'), rawOutput: stdout } };
- Connect to the SSH node. This parses the update list into a newline-separated string.

### 1.5 Add an IF Node to Check for Updates
- **Node**: IF
- **Settings**:
  - Data Type: String
  - Operation: "is not empty"
  - Value 1: {{ $json.updates }
- Connect the true output to the next node; leave false output unconnected (ends workflow if no updates).

### 1.6 Add an HTTP Request Node to Send Notification
- **Node**: HTTP Request
- **Settings**:
  - Method: POST
  - URL: http://10.0.30.2:8123/api/services/notify/mobile_app_sm_s938u (replace with your HA URL and notifier).
  - Authentication: Generic Credential > Header Auth
  - Header Name: Authorization
  - Header Value: Bearer <your-long-lived-token> (replace with your HA token).
  - Headers: Add Content-Type: application/json
  - Send Body: Yes
  - Body Content Type: application/json
  - Body: Expression (fx toggle):
  - Code:
    ```code
    {{ JSON.stringify({ title: "Server Updates Available", message: "The following updates are ready:\n" + ($json.updates || "No updates"), data: { actions: [ { action: "APPROVE_UPDATES", title: "Approve Install" }, { action: "REJECT_UPDATES", title: "Reject" } ] } }) }}
- Connect to the IF node's true output. This sends an actionable notification.

### 1.7 Save and Test
- Save the workflow.
- Run manually to verify updates are detected and a notification is sent.

## Step 2: Set Up the Approval Workflow
This separate workflow handles the "Approve" action and installs updates.

### 2.1 Create the Workflow
Add a new workflow named Server_Update_Approve.

### 2.2 Add a Webhook Trigger
- **Node**: Webhook
- **Settings**:
  - HTTP Method: POST
  - Path: /approve-updates (note the full URL from "Execute Node" for HA config).
  - Respond: Immediately
- This triggers on the HA approval action.

### 2.3 Add an SSH Node to Install Updates
- **Node**: SSH
- **Settings**:
  - Use the same credentials as the check workflow.
  - Command: sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
  - Connect to the Webhook node. This installs updates if approved.

### 2.4 (Optional) Add a Notification
- **Node**: HTTP Request (or reuse the check workflow’s node with adjusted logic).
- **Settings**: Same as Step 1.6, but with a static message like {"message": "Updates installed successfully"}.
- Connect to the SSH node for confirmation.

### 2.5 Save and Test
Save the workflow.
Note the webhook URL (e.g., https://your-n8n.com/webhook/approve-updates).

## Step 3: Configure Home Assistant Automation
This handles the notification actions and calls the webhook.

### 3.1 Add the REST Command
- Edit configuration.yaml
- Add:
  ```yaml
  rest_command:
    approve_updates_webhook:
      url: "https://your-n8n.com/webhook/approve-updates"  # Replace with your webhook URL
      method: post
      content_type: "application/json"
      payload: '{}'
- Restart HA or reload REST commands.

### 3.2 Create the Automation
- In HA, go to Settings > Automations & Scenes > + Create Automation.
- Switch to YAML mode and paste:
  ```yaml
  - alias: "Handle Approve Updates Notification Action"
    trigger:
      - platform: event
        event_type: mobile_app_notification_action
        event_data:
          action: "APPROVE_UPDATES"
    condition: []
    action:
      - service: rest_command.approve_updates_webhook
      - service: notify.mobile_app_sm_s938u  # Replace with your notifier
        data:
          message: "Updates approved. Installing now..."
  - alias: "Handle Reject Updates Notification Action"
    trigger:
      - platform: event
        event_type: mobile_app_notification_action
        event_data:
          action: "REJECT_UPDATES"
    condition: []
    action:
      - service: notify.mobile_app_sm_s938u  # Replace with your notifier
        data:
          message: "Updates rejected. Will check again later."
- Save and reload automations.

## Step 4: Test the Full System
- Run the check workflow manually.
- Tap "Approve" or "Reject" on the notification and verify the outcome (upgrade runs or workflow ends).
- Check n8n (Executions tab) and HA (Logs) for errors.

## Step 5: Duplicate for Other Servers
- Export the check workflow (three-dot menu > Export > Save as JSON).
- Import a new copy, update SSH credentials with the new server’s details, and adjust the HTTP Request URL if using a different HA instance.
- Repeat for each server.

## Notes
- Security: Secure webhook exposure (e.g., VPN or auth) and limit SSH user privileges.
- Customization: Adjust commands for non-Debian servers (e.g., yum for CentOS) and update parsing logic.
- Scaling: Use variables ($env.SERVER_HOST) for dynamic configs.
