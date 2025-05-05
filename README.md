# OBS Queue Dock

A custom browser dock for OBS Studio built with Vue 3 and Vite. This dock allows you to manage a simple checklist or queue, and automatically updates a specified text source in OBS to display the current uncompleted items.

## Features

* Add items to a queue/checklist.
* Remove items from the queue.
* Mark items as complete/incomplete (checked/unchecked).
* Edit item labels directly in the dock.
* Customize the symbol used to indicate the current top item.
* Automatically updates a designated OBS text source with the list of *incomplete* items.
* The top incomplete item is highlighted with a symbol (default: ▶).
* Smooth transitions when adding/removing items.

## Prerequisites

Before you begin, ensure you have the following installed:

* **[Node.js](https://nodejs.org/)**
  - (LTS version recommended) Includes npm.
* **[OBS Studio](https://obsproject.com/)**
  - The streaming software itself.
* **[OBS WebSocket Plugin](https://github.com/obsproject/obs-websocket/releases)**
  - Version 5.x or later is required.

## Setup Instructions

Follow these steps to get the OBS Queue Dock running locally:

**1. Clone the Repository:**
```
git clone https://github.com/ltrademark/Queue-Dock-for-OBS.git
cd queue-dock-for-obs
```

**2. Install Dependencies**

Using npm
`npm install`
Or using Yarn
`yarn install`

**3. Configure OBS WebSocket**

- Install the OBS WebSocket plugin if you haven't already.
- In OBS Studio, go to Tools -> WebSocket Server Settings.
- Enable the WebSocket server.Note the Server Port (default is 4455).
- (Recommended) Enable Authentication and set a secure Server Password. Remember this password.

**4. Create an OBS Text Source**

- In the OBS scene where you want to display the queue, add a new source: Sources -> + -> Text (GDI+) (or your preferred text source type).
- Give it a memorable name (e.g., QueueDisplay). You'll need this exact name later.
- You can leave the default text empty for now; the dock will populate it. Customize font, size, and color as desired.

**5. Set Up Environment Variables**

Create a file named .env in the root directory of the cloned project (queue-dock-for-obs/.env).Add the following lines to the .env file, replacing the placeholder values with your actual OBS settings

# .env

```bash
# URL for the OBS WebSocket server (usually localhost on the default port)
VITE_OBS_URL=ws://localhost:4455

# The password you set in OBS WebSocket Server Settings (leave blank if authentication is disabled)
VITE_OBS_PASSWORD=your_obs_websocket_password

# The exact name of the Text Source you created in OBS
VITE_OBS_TEXT_SOURCE=QueueDisplay
```

> Important: **Make sure your .gitignore file includes .env to prevent accidentally committing your password.**

## Running the Dock

**1. Start the Development Server**
  - `npm run dev` or `yarn dev`
    - This will start the Vite development server. Look for a message in your terminal indicating the local URL, usually http://localhost:5173 (the port might vary).

**2. Add the Custom Browser Dock in OBS**
  - Copy the local URL provided by Vite (e.g., http://localhost:5173).
  - In OBS Studio, go to `Docks` -> `Custom Browser Docks`
  - Enter a name for the dock (e.g., Queue Manager) under Dock Name.
  - Paste the local URL into the URL field next to it.
  - Click `Apply`, then `Close`.
  - A new dock window named "Queue Manager" should appear. You can drag and attach it to your OBS layout like any other dock.

# How to Use

- **UsageConnect**
  - The dock should automatically attempt to connect to OBS WebSocket when it loads. Check your browser's developer console (usually F12) within the dock window if you encounter connection issues (Right-click dock -> Inspect).
- **Add Items**
  - Type a task into the input field and press Enter or click the `+` button. New items appear at the top.
- **Complete Items**
  - Click the checkbox next to an item to mark it as complete. Completed items are visually distinct (strikethrough) and won't appear in the OBS text source.
- **Edit Items**
  - Click directly on the text label of an item to edit it. Press Enter or click away to save the change.
- **Remove Items**
  - Click the `❌` button next to an item to remove it.
- **Change Symbol**
  - Edit the symbol in the top-right input field to change the indicator for the current top item in the OBS text source.
- **OBS Text Source**
  - The designated text source in OBS will automatically update to show the list of incomplete items, with the top item prefixed by your chosen symbol.Contributing

> *(Optional: Add contribution guidelines if you welcome contributions)Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.*
