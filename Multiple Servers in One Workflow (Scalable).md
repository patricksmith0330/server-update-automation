## Using a Loop for Multiple Servers in One n8n Workflow
This guide explains how to configure a single n8n workflow to handle multiple servers by using a list of servers as input and looping over them. This avoids duplication and scales well for checking APT updates across servers

### Prerequisites
- n8n installed and running (version 1.x recommended).
- SSH access to multiple Debian/Ubuntu-based servers.
- The base workflow for APT update checks (SSH, Code, IF, etc.) already set up.

## Step 1: Add a Function Node for Server List
- After the Schedule trigger, add a Function node (mode: "Run Once for All Items").
- Code:
  ```javascript
  // Define your servers as an array of objects
  return [
    { json: { host: "10.0.30.2", user: "user1", privateKey: "key1" } },
    { json: { host: "10.0.30.3", user: "user2", privateKey: "key2" } }
    // Add more servers as needed
  ];
- This outputs multiple items (one per server).

### Step 2: Update the SSH Node
- Connect to the Function node.
- In credentials:
  - Host: {{ $json.host }}
  - Username: {{ $json.user }}
  - Private Key: {{ $json.privateKey }}
- The workflow will now run the SSH check for each server item.

### Step 3: Loop the Rest of the Workflow
- The Code, IF, Set HA Entity, and Notification nodes will process each server's data separately (e.g., separate entities in HA like sensor.server1_updates, sensor.server2_updates).
- Update the "Set HA Entity" body to dynamic entity_id, e.g.:
- Code:
  ```text
  {{ JSON.stringify({ entity_id: "sensor.server_" + $json.host.replace('.', '_') + "_updates", long_value: $json.updates }) }}
- Create corresponding sensors in HA for each server.

### Step 4: For Approval
- In the dashboard, add buttons per server, calling separate webhooks (e.g., /approve-server1).
- Duplicate the approval workflow for each, or parameterize with $json.host if looping.

### Notes
- Testing: Run with one server, then scale.
- Errors: If variables don't resolve, check n8n logs for env loading.
- Scaling: For many servers, use a database node to fetch a list dynamically.
