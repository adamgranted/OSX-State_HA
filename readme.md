
# OSX Screen State for Home Assistant
This is a simple configuration to report the screen lock state and screesaver state of macOS back to Home Assistant. It requires [Hammersppon](https://github.com/Hammerspoon/hammerspoon) and a long lived HA token. Hammerspoon handles the state tracking of OSX and provides the necessary hooks we need to report back to HA.

I'm open to contributions if there's a better way to approach this. Lock/Unlock OSX hooks are primarily deprecated at this point, this seemed to be the best alternative.

## Prerequisites

1. Install Hammerspoon ([see instructions](https://github.com/Hammerspoon/hammerspoon?tab=readme-ov-file#how-do-i-install-it)) and **give it Accessibility permissions**. 
2. Obtain a [Long-Lived Access Token](https://developers.home-assistant.io/docs/auth_api/#long-lived-access-token) in Home Assistant.


## HA Configuration
1. In your `configuration.yml`, add the following:
   
   ```
    homeassistant:
       customize:
           binary_sensor.macos_lock_state:
           friendly_name: "macOS Lock State"
           device_class: lock  # shows lock/unlock icon
           binary_sensor.macos_screensaver:
           friendly_name: "macOS Screensaver"
           icon: mdi:television
   ```
   
3. Restart HA

## Hammerspoon Configuration

1. Enable "Start at Login" for Hammerspoon. *Preferences > “Launch Hammerspoon at login."*
2. Open Hammerspoon’s config file. Click the Hammerspoon menu bar icon > Open Config or create/edit `~/.hammerspoon/init.lua`.
3. Paste the following code, adjusting your HA server instance and credentials:
   
    ```
    ------------------------------------------------
    -- 1. Home Assistant configuration
    ------------------------------------------------
    local HA_URL   = "http://YOUR_HA_IP_OR_DOMAIN:8123"  -- or https:// if using SSL
    local HA_TOKEN = "PASTE_YOUR_LONG_LIVED_TOKEN_HERE"

    -- We'll use two entity_ids:
    local LOCK_ENTITY       = "binary_sensor.macos_lock_state"
    local SCREENSAVER_ENTITY = "binary_sensor.macos_screensaver"

    -- Helper to post "on"/"off" to a binary_sensor in HA
    local function postBinarySensorStateToHA(entity_id, isOn)
    local endpoint = HA_URL .. "/api/states/" .. entity_id
    local headers  = {
        ["Authorization"] = "Bearer " .. HA_TOKEN,
        ["Content-Type"]  = "application/json"
    }
    -- HA interprets `state = "on"` or "off" for a binary_sensor
    local stateStr = isOn and "on" or "off"
    local body     = hs.json.encode({ state = stateStr })

    hs.http.asyncPost(endpoint, body, headers, function(status, responseBody, responseHeaders)
        print("POST " .. entity_id .. " => " .. stateStr .. " (status: " .. tostring(status) .. ")")
    end)
    end

    ------------------------------------------------
    -- 2. Watch for lock/unlock & screensaver events
    ------------------------------------------------
    -- We'll use hs.caffeinate.watcher for these events:
    --   screensDidLock       => locked
    --   screensDidUnlock     => unlocked
    --   screensaverDidStart  => screensaver on
    --   screensaverDidStop   => screensaver off

    local function lockWatcherCallback(eventType)
    if eventType == hs.caffeinate.watcher.screensDidLock then
        -- The Mac is locked (password required)
        postBinarySensorStateToHA(LOCK_ENTITY, true)   -- on = locked

    elseif eventType == hs.caffeinate.watcher.screensDidUnlock then
        -- The Mac is unlocked
        postBinarySensorStateToHA(LOCK_ENTITY, false)  -- off = unlocked

    elseif eventType == hs.caffeinate.watcher.screensaverDidStart then
        -- Screensaver just started
        postBinarySensorStateToHA(SCREENSAVER_ENTITY, true)   -- on = screensaver is active

    elseif eventType == hs.caffeinate.watcher.screensaverDidStop then
        -- Screensaver stopped
        postBinarySensorStateToHA(SCREENSAVER_ENTITY, false)  -- off = screensaver inactive
    end
    end

    -- Create and start the watcher
    lockWatcher = hs.caffeinate.watcher.new(lockWatcherCallback)
    lockWatcher:start()

    ------------------------------------------------
    -- 3. Initialize states on Hammerspoon start
    ------------------------------------------------
    -- We'll assume the Mac is unlocked and screensaver is off
    postBinarySensorStateToHA(LOCK_ENTITY, false)       -- unlocked
    postBinarySensorStateToHA(SCREENSAVER_ENTITY, false)  -- inactive
    ```


## Notes
- HA tokens will be stored in plain text. This could be moved to keychain with some more effort.
- If your mac is set to lock as soon as the screensaver starts, you might see “screensaver -> locked” transitions quickly. This is easily handled within an automation however.
