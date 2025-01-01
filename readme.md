
# macOS Screen State for Home Assistant
This is a simple configuration to report the screen lock state, screesaver state, and USB dock connection state of macOS back to Home Assistant. It requires [Hammerspoon](https://github.com/Hammerspoon/hammerspoon) and a long lived HA token. Hammerspoon handles the state tracking of OSX and provides the necessary hooks we need to report back to HA.

I'm open to contributions if there's a better way to approach this. Lock/Unlock OSX hooks are primarily deprecated at this point, this seemed to be the best alternative.

## Prerequisites

1. Install Hammerspoon ([see instructions](https://github.com/Hammerspoon/hammerspoon?tab=readme-ov-file#how-do-i-install-it)) and **give it Accessibility permissions**. 
2. Obtain a [Long-Lived Access Token](https://developers.home-assistant.io/docs/auth_api/#long-lived-access-token) in Home Assistant.
   - Click on your profile icon from HA
   - Go to security
   - Create a long lived access token and give it a name


## HA Configuration
1. In your `configuration.yml`, add the following:
   
   ```
    homeassistant:
        customize:
            binary_sensor.macos_lock_state:
            friendly_name: "macOS Lock State"
            device_class: lock  # 'on' => Locked, 'off' => Unlocked

            binary_sensor.macos_screensaver:
            friendly_name: "macOS Screensaver"
            icon: mdi:television  # or another icon

            binary_sensor.macos_dock_connected:
            friendly_name: "Dock"
            device_class: connectivity  # 'on' => Connected, 'off' => Disconnected
   ```
   
3. Restart HA

**NOTE: The entity will not appear in HA until after the state past being posted via the REST API**

## Hammerspoon Configuration

1. Enable "Start at Login" for Hammerspoon. *Preferences > “Launch Hammerspoon at login."*
2. Open Hammerspoon’s config file. Click the Hammerspoon menu bar icon > Open Config or create/edit `~/.hammerspoon/init.lua`.
3. Paste the following code, adjusting your HA server instance and credentials:
   
    ```
    ------------------------------------------------
    -- 1. Home Assistant configuration
    ------------------------------------------------
    local HA_URL   = "http://YOUR_HA_IP_OR_DOMAIN:8123"  -- or https:// if using SSL
    local HA_TOKEN = "YOUR_LONG_LIVED_TOKEN"

    -- We'll use three entity_ids:
    local LOCK_ENTITY        = "binary_sensor.macos_lock_state"
    local SCREENSAVER_ENTITY = "binary_sensor.macos_screensaver"
    local DOCK_ENTITY        = "binary_sensor.macos_dock_connected"  -- for the Thunderbolt dock
    local function postBinarySensorStateToHA(entity_id, isOn)
        local endpoint = HA_URL .. "/api/states/" .. entity_id
        local headers  = {
            ["Authorization"] = "Bearer " .. HA_TOKEN,
            ["Content-Type"]  = "application/json"
        }
        -- HA interprets `state` = "on" or "off" for a binary_sensor
        local stateStr = isOn and "on" or "off"
        local body     = hs.json.encode({ state = stateStr })

        hs.http.asyncPost(endpoint, body, headers, function(status, responseBody, responseHeaders)
            print("POST " .. entity_id .. " => " .. stateStr .. " (status: " .. tostring(status) .. ")")
        end)
    end

    ------------------------------------------------
    -- 2A. Watch for lock/unlock & screensaver events
    ------------------------------------------------
    local function lockWatcherCallback(eventType)
        if eventType == hs.caffeinate.watcher.screensDidLock then
            -- The Mac is locked (password required)
            postBinarySensorStateToHA(LOCK_ENTITY, true)   -- on = locked

        elseif eventType == hs.caffeinate.watcher.screensDidUnlock then
            -- The Mac is unlocked
            postBinarySensorStateToHA(LOCK_ENTITY, false)  -- off = unlocked

        elseif eventType == hs.caffeinate.watcher.screensaverDidStart then
            -- Screensaver just started
            postBinarySensorStateToHA(SCREENSAVER_ENTITY, true)   -- on = screensaver active

        elseif eventType == hs.caffeinate.watcher.screensaverDidStop then
            -- Screensaver stopped
            postBinarySensorStateToHA(SCREENSAVER_ENTITY, false)  -- off = screensaver inactive
        end
    end

    lockWatcher = hs.caffeinate.watcher.new(lockWatcherCallback)
    lockWatcher:start()

    ------------------------------------------------
    -- 2B. Watch for Thunderbolt Dock via USB (by vendorID/productID)
    ------------------------------------------------

    -- Replace these with the IDs you found for your dock
    local VENDOR_ID  = 1234   -- or 0x1234 if you have hex
    local PRODUCT_ID = 5678 

    local function usbCallback(device)
        if device.vendorID == VENDOR_ID and device.productID == PRODUCT_ID then
            if device.eventType == "added" then
                print("Dock connected!")
                postBinarySensorStateToHA(DOCK_ENTITY, true)  -- on => dock connected
            elseif device.eventType == "removed" then
                print("Dock disconnected!")
                postBinarySensorStateToHA(DOCK_ENTITY, false) -- off => dock disconnected
            end
        end
    end

    usbWatcher = hs.usb.watcher.new(usbCallback)
    usbWatcher:start()

    ------------------------------------------------
    -- 3. Initialize states on Hammerspoon start
    ------------------------------------------------
    -- If you want to detect the dock at startup, you can do:
    local alreadyConnected = false
    for _, d in ipairs(hs.usb.attachedDevices()) do
        if d.vendorID == VENDOR_ID and d.productID == PRODUCT_ID then
            alreadyConnected = true
            break
        end
    end

    postBinarySensorStateToHA(LOCK_ENTITY, false)        -- assume unlocked
    postBinarySensorStateToHA(SCREENSAVER_ENTITY, false) -- assume screensaver off
    postBinarySensorStateToHA(DOCK_ENTITY, alreadyConnected)
    ```


## Notes
- HA tokens will be stored in plain text. This could be moved to keychain with some more effort.
- If your mac is set to lock as soon as the screensaver starts, you might see “screensaver -> locked” transitions quickly. This is easily handled within an automation however.
- Reporting could be moved to MQTT, the HA API is working fine as is however.
