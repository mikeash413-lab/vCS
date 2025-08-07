// vCS Roleplay Module v1.2 - Fixed & Optimized
// Enhanced roleplay mechanics with proper health core integration

// Global Variables
key owner_key;
key captor_key = NULL_KEY;
float escape_chance = 0.15;
integer escape_attempts = 0;
integer max_escape_attempts = 5;
float last_capture_time = 0.0;
integer capture_timeout = 300;

// Status from Health Core (don't track locally to avoid desync)
integer is_unconscious = FALSE;
integer is_captured = FALSE;
integer current_mode = 1;

// Mode System
integer MODE_COMBAT = 1;
integer MODE_ROLEPLAY = 2; 
integer MODE_STAFF = 3;

// Replace with actual creator key - this is currently NULL_KEY
key CREATOR_KEY = "PUT_YOUR_ACTUAL_UUID_HERE";
integer is_creator = FALSE;

list roleplay_emotes = [
    "struggles against their bonds",
    "looks around nervously", 
    "tries to break free",
    "calls for help",
    "remains defiant",
    "shows fear in their eyes",
    "attempts to negotiate",
    "pleads for release",
    "tests the strength of their bonds",
    "searches for a way to escape",
    "whispers quietly to themselves",
    "takes a deep breath to stay calm"
];

// Channels
integer ROLEPLAY_CHANNEL = -7;
integer HUD_CHANNEL = -402;

// Input sanitization
list allowed = ["A","B","C","D","E","F","G","H","I","J","K","L","M","N","O","P","Q","R","S","T","U","V","W","X","Y","Z",
                "a","b","c","d","e","f","g","h","i","j","k","l","m","n","o","p","q","r","s","t","u","v","w","x","y","z",
                "0","1","2","3","4","5","6","7","8","9"," ","-","_"];

string sanitize(string input, integer maxlen) {
    if (llStringLength(input) > maxlen) input = llGetSubString(input, 0, maxlen-1);
    integer i; string out = "";
    for (i = 0; i < llStringLength(input); ++i) {
        string c = llGetSubString(input, i, i);
        if (llListFindList(allowed, [c]) != -1) out += c;
    }
    return out;
}

// Request current status from health core
requestStatusUpdate() {
    llMessageLinked(LINK_SET, 0, "GET_STATUS", "");
}

sendRoleplayEmote() {
    if (is_captured) {
        string emote = llList2String(roleplay_emotes, llFloor(llFrand(llGetListLength(roleplay_emotes))));
        llRegionSay(ROLEPLAY_CHANNEL, "/me " + emote);
    }
}

handleEscapeAttempt() {
    if (!is_captured) {
        llOwnerSay("You are not captured!");
        return;
    }
    
    escape_attempts++;
    if (escape_attempts >= max_escape_attempts) {
        llOwnerSay("You've exhausted your escape attempts (" + (string)max_escape_attempts + "/" + (string)max_escape_attempts + "). You must wait for release or rescue.");
        return;
    }
    
    float roll = llFrand(1.0);
    if (roll <= escape_chance) {
        // Successfully escaped - notify health core
        llMessageLinked(LINK_SET, 0, "ESCAPE_SUCCESS", "");
        llRegionSay(ROLEPLAY_CHANNEL, "/me breaks free from their bonds!");
        llOwnerSay("You successfully escaped!");
        
        // Notify captor
        if (captor_key != NULL_KEY) {
            llRegionSayTo(captor_key, ROLEPLAY_CHANNEL, sanitize(llKey2Name(owner_key), 32) + " has escaped!");
        }
        
        // Reset local tracking
        captor_key = NULL_KEY;
        escape_attempts = 0;
        llSetTimerEvent(0.0); // Stop timer
    } else {
        sendRoleplayEmote();
        integer remaining = max_escape_attempts - escape_attempts;
        llOwnerSay("Escape attempt " + (string)escape_attempts + "/" + (string)max_escape_attempts + " failed. " + (string)remaining + " attempts remaining.");
    }
}

checkCaptureTimeout() {
    if (is_captured && (llGetTime() - last_capture_time) > capture_timeout) {
        // Timeout escape - notify health core
        llMessageLinked(LINK_SET, 0, "ESCAPE_TIMEOUT", "");
        llRegionSay(ROLEPLAY_CHANNEL, "/me manages to work free after a long struggle.");
        llOwnerSay("Automatic release after " + (string)(capture_timeout/60) + " minutes.");
        
        // Reset local tracking
        captor_key = NULL_KEY;
        escape_attempts = 0;
        llSetTimerEvent(0.0);
    }
}

setMode(integer new_mode) {
    if (new_mode == current_mode) return;
    
    // Check staff mode permissions
    if (new_mode == MODE_STAFF && !is_creator) {
        llOwnerSay("⚠️ Staff Mode is restricted to the creator only");
        return;
    }
    
    current_mode = new_mode;
    
    // Notify health core of mode change
    llMessageLinked(LINK_SET, current_mode, "MODE_CHANGE", "");
    
    // Notify other systems via channel
    llRegionSay(HUD_CHANNEL, "MODE:" + (string)current_mode);
    
    string mode_name;
    if (current_mode == MODE_COMBAT) mode_name = "⚔️ Combat Mode";
    else if (current_mode == MODE_ROLEPLAY) mode_name = "🎭 Roleplay Mode (Damage Immunity)";
    else if (current_mode == MODE_STAFF) mode_name = "👑 Staff Mode (Unlimited Powers)";
    
    llOwnerSay("Roleplay Module: " + mode_name + " activated");
}

startCapture(key captor) {
    captor_key = captor;
    escape_attempts = 0;
    last_capture_time = llGetTime();
    
    string captor_name = sanitize(llKey2Name(captor), 32);
    
    // Roleplay announcements
    llRegionSay(ROLEPLAY_CHANNEL, "/me is bound and captured by " + captor_name);
    llOwnerSay("🔒 You have been captured by " + captor_name);
    llOwnerSay("💡 Commands: '/7 escape' (" + (string)((integer)(escape_chance*100)) + "% chance, " + (string)max_escape_attempts + " max attempts)");
    llOwnerSay("⏰ Maximum capture time: " + (string)(capture_timeout/60) + " minutes");
    
    // Notify captor
    llRegionSayTo(captor, ROLEPLAY_CHANNEL, "🔒 You captured " + sanitize(llKey2Name(owner_key), 32) + ".");
    llRegionSayTo(captor, ROLEPLAY_CHANNEL, "💡 Say '/7 release' to free them, or they can attempt escape.");
    
    // Start capture timer
    llSetTimerEvent(60.0);
}

default
{
    state_entry()
    {
        owner_key = llGetOwner();
        is_creator = (owner_key == CREATOR_KEY);
        
        // Listen to roleplay channel
        llListen(ROLEPLAY_CHANNEL, "", "", "");
        
        // Request initial status from health core
        requestStatusUpdate();
        
        llOwnerSay("vCS Roleplay Module v1.2 initialized");
        if (CREATOR_KEY == "PUT_YOUR_ACTUAL_UUID_HERE") {
            llOwnerSay("⚠️ Warning: CREATOR_KEY not set - Staff mode will not work");
        }
        
        // Set initial mode
        setMode(current_mode);
    }
    
    listen(integer channel, string name, key id, string message)
    {
        if (channel == ROLEPLAY_CHANNEL) {
            // Owner commands
            if (id == owner_key || llGetOwnerKey(id) == owner_key) {
                string msg = llToLower(sanitize(message, 32));
                
                if (msg == "combat") {
                    setMode(MODE_COMBAT);
                }
                else if (msg == "roleplay") {
                    setMode(MODE_ROLEPLAY);
                }
                else if (msg == "staff") {
                    if (is_creator) {
                        setMode(MODE_STAFF);
                    } else {
                        llOwnerSay("⚠️ Staff Mode requires creator permissions");
                    }
                }
                else if (msg == "escape") {
                    handleEscapeAttempt();
                }
                else if (msg == "emote") {
                    sendRoleplayEmote();
                }
                else if (msg == "status") {
                    requestStatusUpdate();
                    string mode_name;
                    if (current_mode == MODE_COMBAT) mode_name = "Combat";
                    else if (current_mode == MODE_ROLEPLAY) mode_name = "Roleplay";
                    else mode_name = "Staff";
                    llOwnerSay("📊 Status: Mode=" + mode_name + " | Unconscious=" + (string)is_unconscious + " | Captured=" + (string)is_captured);
                    if (is_captured) {
                        float time_left = capture_timeout - (llGetTime() - last_capture_time);
                        llOwnerSay("⏰ Capture time remaining: " + (string)((integer)(time_left/60)) + " minutes");
                        llOwnerSay("🔓 Escape attempts: " + (string)escape_attempts + "/" + (string)max_escape_attempts);
                    }
                }
            }
            // Other player commands
            else {
                if (message == "capture" && is_unconscious && !is_captured) {
                    vector my_pos = llGetPos();
                    list details = llGetObjectDetails(id, [OBJECT_POS]);
                    if (details != []) {
                        vector their_pos = llList2Vector(details, 0);
                        float distance = llVecDist(my_pos, their_pos);
                        if (distance <= 5.0) {
                            // Notify health core of capture
                            llMessageLinked(LINK_SET, 0, "CAPTURE_START", (string)id);
                            startCapture(id);
                        } else {
                            llRegionSayTo(id, ROLEPLAY_CHANNEL, "❌ Too far away to capture. Get within 5m (currently " + (string)((integer)distance) + "m away).");
                        }
                    }
                }
                else if (message == "release" && is_captured && id == captor_key) {
                    // Notify health core of release
                    llMessageLinked(LINK_SET, 0, "RELEASE", "");
                    
                    llRegionSay(ROLEPLAY_CHANNEL, "/me is released by their captor");
                    llOwnerSay("🔓 You have been released!");
                    llRegionSayTo(id, ROLEPLAY_CHANNEL, "✅ You released " + sanitize(llKey2Name(owner_key), 32));
                    
                    // Reset local tracking
                    captor_key = NULL_KEY;
                    escape_attempts = 0;
                    llSetTimerEvent(0.0);
                }
            }
        }
    }
    
    // Handle messages from health core
    link_message(integer sender_num, integer num, string str, key id)
    {
        if (str == "STATUS_RESPONSE") {
            // Parse status from health core: "unconscious:captured:mode"
            list status_parts = llParseString2List((string)id, [":"], []);
            if (llGetListLength(status_parts) >= 2) {
                integer new_unconscious = (integer)llList2String(status_parts, 0);
                integer new_captured = (integer)llList2String(status_parts, 1);
                
                // Update local cache
                is_unconscious = new_unconscious;
                is_captured = new_captured;
                
                // If we're no longer captured, reset tracking
                if (!is_captured && captor_key != NULL_KEY) {
                    captor_key = NULL_KEY;
                    escape_attempts = 0;
                    llSetTimerEvent(0.0);
                }
                
                // If we just got captured, start capture tracking
                if (is_captured && captor_key == NULL_KEY) {
                    // Health core should provide captor key
                    if (llGetListLength(status_parts) >= 3) {
                        captor_key = (key)llList2String(status_parts, 2);
                        last_capture_time = llGetTime();
                        llSetTimerEvent(60.0);
                    }
                }
            }
        }
        else if (str == "MODE_CHANGE") {
            current_mode = num;
        }
    }
    
    timer()
    {
        // Check for capture timeout
        checkCaptureTimeout();
        
        // Send random emotes while captured
        if (is_captured && llFrand(1.0) < 0.3) {
            sendRoleplayEmote();
        }
        
        // Periodically sync with health core
        if (llFrand(1.0) < 0.1) {
            requestStatusUpdate();
        }
    }
    
    touch_start(integer total_number)
    {
        key toucher = llDetectedKey(0);
        if (toucher == owner_key) {
            llOwnerSay("🎭 vCS Roleplay Module v1.2\n" +
                      "━━━━━━━━━━━━━━━━━━━━━━━\n" +
                      "'/7 combat' - Switch to combat mode\n" +
                      "'/7 roleplay' - Switch to roleplay mode\n" +
                      "'/7 staff' - Staff mode (creator only)\n" +
                      "'/7 escape' - Attempt escape (when captured)\n" +
                      "'/7 emote' - Send random roleplay emote\n" +
                      "'/7 status' - Show current status\n" +
                      "━━━━━━━━━━━━━━━━━━━━━━━");
        }
    }
}
