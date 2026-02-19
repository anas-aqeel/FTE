# WhatsApp Group Identification Strategy

**Last Updated:** 2026-02-18
**Decision:** Use WhatsApp Group ID (Not Name)

---

## Problem Statement

WhatsApp groups in the allowlist need a stable identifier for matching. Two options:

1. **Group Name** - Human-readable but can change at any time
2. **Group ID** - Stable, unique identifier that never changes

---

## Decision: Use Group ID

### Rationale:

✅ **Stability** - Group IDs never change, even if the group is renamed
✅ **Uniqueness** - IDs are globally unique across WhatsApp
✅ **Reliability** - Prevents allowlist breakage when groups are renamed
✅ **API Support** - whatsapp-web.js provides stable `chat.id._serialized`

❌ **Group names can change** - Admins can rename groups at any time, breaking name-based matching

---

## Implementation

### 1. Database Schema

**user_settings table:**

```sql
whatsapp_group_allowlist: JSONB

-- Format:
{
  "groups": [
    {
      "id": "1234567890@g.us",           -- Stable WhatsApp group ID
      "name": "CS101 Project Team",       -- Current name (for display)
      "added_at": "2024-03-01T10:00:00Z",
      "updated_at": "2024-03-01T10:00:00Z"
    },
    {
      "id": "9876543210@g.us",
      "name": "Database Study Group",
      "added_at": "2024-03-05T15:30:00Z",
      "updated_at": "2024-03-05T15:30:00Z"
    }
  ]
}
```

**Why JSONB array of objects?**
- Stores both ID (for matching) and name (for display)
- Tracks when group was added to allowlist
- Allows updating name when group is renamed

---

### 2. WhatsApp Service - Group Detection

**File:** `whatsapp-service/src/messageHandler.js`

```javascript
const whatsapp = require('whatsapp-web.js');

class MessageHandler {
  constructor(client, allowlistConfig) {
    this.client = client;
    this.allowlist = allowlistConfig.groups || [];
    console.log(`Loaded allowlist with ${this.allowlist.length} groups`);
  }

  async handleIncomingMessage(message) {
    const chat = await message.getChat();

    // Get stable identifiers
    const chatId = chat.id._serialized;  // e.g., "1234567890@g.us"
    const chatName = chat.name;
    const isGroup = chat.isGroup;

    console.log(`Message from: ${chatName} (${chatId}), isGroup: ${isGroup}`);

    // Determine if should buffer this chat
    const shouldBuffer = isGroup
      ? this.isAllowlisted(chatId)
      : true;  // All private chats always buffered

    if (shouldBuffer) {
      this.bufferMessage({
        chatId: chatId,
        chatName: chatName,
        isGroup: isGroup,
        sender: message.author || message.from,
        body: message.body,
        timestamp: message.timestamp,
        hasMedia: message.hasMedia
      });
    } else {
      console.log(`Skipping non-allowlisted group: ${chatName}`);
    }
  }

  isAllowlisted(chatId) {
    // Match by ID (primary method)
    const isAllowed = this.allowlist.some(group => group.id === chatId);
    console.log(`Group ${chatId} allowlisted: ${isAllowed}`);
    return isAllowed;
  }

  updateAllowlist(newAllowlist) {
    // Called when allowlist changes (via API)
    this.allowlist = newAllowlist.groups || [];
    console.log(`Allowlist updated: ${this.allowlist.length} groups`);
  }

  // Helper: Get all groups user is in (for allowlist setup)
  async getAllGroups() {
    const chats = await this.client.getChats();
    return chats
      .filter(chat => chat.isGroup)
      .map(chat => ({
        id: chat.id._serialized,
        name: chat.name,
        participants: chat.participants.length,
        description: chat.description || ''
      }));
  }
}

module.exports = MessageHandler;
```

**Key Points:**
- `chat.id._serialized` gives the stable group ID (format: `<number>@g.us`)
- Private chats use format: `<number>@c.us`
- Matching is done by ID, not name

---

### 3. Backend - Allowlist Management Endpoints

**File:** `backend/app/routers/whatsapp_settings.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from typing import List
import requests
from datetime import datetime

router = APIRouter(prefix="/whatsapp", tags=["WhatsApp Settings"])


@router.get("/groups")
async def list_available_groups(current_user: User = Depends(get_current_user)):
    """
    Query WhatsApp service for all groups user is in.
    Used to populate allowlist selection UI.
    """
    try:
        response = requests.get(
            f"{WHATSAPP_SERVICE_URL}/api/groups",
            headers={"X-API-Key": WHATSAPP_API_KEY},
            timeout=10
        )
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        raise HTTPException(status_code=503, detail=f"WhatsApp service unavailable: {e}")


@router.get("/allowlist")
async def get_current_allowlist(current_user: User = Depends(get_current_user)):
    """Get user's current WhatsApp group allowlist"""
    settings = supabase.table('user_settings') \
        .select('whatsapp_group_allowlist') \
        .eq('user_id', current_user.id) \
        .execute()

    if not settings.data:
        return {"groups": []}

    return settings.data[0]['whatsapp_group_allowlist'] or {"groups": []}


@router.post("/allowlist")
async def update_group_allowlist(
    group_ids: List[str],
    current_user: User = Depends(get_current_user)
):
    """
    Update user's WhatsApp group allowlist.
    Accepts list of group IDs (e.g., ["1234567890@g.us", "9876543210@g.us"])
    """
    # Fetch current group details from WhatsApp service
    try:
        groups_info = await fetch_group_details(group_ids)
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Failed to fetch group details: {e}")

    # Build allowlist structure
    allowlist = {
        "groups": [
            {
                "id": g["id"],
                "name": g["name"],
                "added_at": datetime.now().isoformat(),
                "updated_at": datetime.now().isoformat()
            }
            for g in groups_info
        ]
    }

    # Update user_settings
    supabase.table('user_settings').upsert({
        'user_id': current_user.id,
        'whatsapp_group_allowlist': allowlist,
        'updated_at': datetime.now().isoformat()
    }).execute()

    # Notify WhatsApp service of allowlist change
    try:
        requests.post(
            f"{WHATSAPP_SERVICE_URL}/api/allowlist",
            json={"allowlist": allowlist},
            headers={"X-API-Key": WHATSAPP_API_KEY},
            timeout=10
        )
    except requests.exceptions.RequestException as e:
        # Log warning but don't fail - allowlist is saved in DB
        logger.warning(f"Failed to notify WhatsApp service: {e}")

    return {
        "status": "updated",
        "groups": len(allowlist["groups"]),
        "allowlist": allowlist
    }


@router.post("/group-update")
async def handle_group_update(
    update: GroupUpdate,
    api_key: str = Depends(verify_whatsapp_api_key)
):
    """
    Called by WhatsApp service when a group is renamed.
    Updates group name in all users' allowlists.
    """
    # Find all users who have this group in their allowlist
    all_settings = supabase.table('user_settings').select('*').execute()

    updated_count = 0
    for user_settings in all_settings.data:
        allowlist = user_settings.get('whatsapp_group_allowlist')

        if not allowlist or 'groups' not in allowlist:
            continue

        # Check if this group is in the allowlist
        for group in allowlist['groups']:
            if group['id'] == update.group_id:
                # Update name and timestamp
                group['name'] = update.new_name
                group['updated_at'] = datetime.now().isoformat()

                # Save updated allowlist
                supabase.table('user_settings').update({
                    'whatsapp_group_allowlist': allowlist,
                    'updated_at': datetime.now().isoformat()
                }).eq('id', user_settings['id']).execute()

                updated_count += 1
                break

    return {
        "status": "updated",
        "group_id": update.group_id,
        "new_name": update.new_name,
        "users_updated": updated_count
    }


async def fetch_group_details(group_ids: List[str]) -> List[Dict]:
    """Fetch group names from WhatsApp service for given IDs"""
    response = requests.post(
        f"{WHATSAPP_SERVICE_URL}/api/groups/details",
        json={"group_ids": group_ids},
        headers={"X-API-Key": WHATSAPP_API_KEY},
        timeout=10
    )
    response.raise_for_status()
    return response.json()


# Models
from pydantic import BaseModel

class GroupUpdate(BaseModel):
    group_id: str
    new_name: str
```

---

### 4. WhatsApp Service - Group Name Change Tracking

**File:** `whatsapp-service/src/eventHandlers.js`

```javascript
class EventHandlers {
  constructor(client, backendClient) {
    this.client = client;
    this.backendClient = backendClient;
  }

  setupHandlers() {
    // Monitor group info updates (name changes)
    this.client.on('group_update', async (notification) => {
      if (notification.type === 'subject') {  // Subject = group name
        await this.handleGroupNameChange(notification);
      }
    });
  }

  async handleGroupNameChange(notification) {
    try {
      const chat = await notification.getChat();
      const chatId = chat.id._serialized;
      const newName = notification.body;  // New group name

      console.log(`Group ${chatId} renamed to: ${newName}`);

      // Notify backend of name change
      await this.backendClient.post('/whatsapp/group-update', {
        group_id: chatId,
        new_name: newName
      });

      console.log(`Backend notified of group rename: ${chatId} -> ${newName}`);
    } catch (error) {
      console.error('Failed to handle group name change:', error);
    }
  }
}

module.exports = EventHandlers;
```

**What happens when a group is renamed:**
1. WhatsApp service detects `group_update` event
2. Extracts new name and stable group ID
3. Notifies backend via `POST /whatsapp/group-update`
4. Backend updates `name` field in all users' allowlists
5. **Matching continues to work** because it's based on ID, not name

---

### 5. WhatsApp Service - API Endpoints

**File:** `whatsapp-service/src/api.js`

```javascript
const express = require('express');
const router = express.Router();

// Get all groups user is in
router.get('/groups', async (req, res) => {
  try {
    const groups = await messageHandler.getAllGroups();
    res.json(groups);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get details for specific group IDs
router.post('/groups/details', async (req, res) => {
  try {
    const { group_ids } = req.body;
    const chats = await client.getChats();

    const details = group_ids.map(id => {
      const chat = chats.find(c => c.id._serialized === id);
      if (!chat) {
        return { id, error: 'Group not found' };
      }
      return {
        id: chat.id._serialized,
        name: chat.name,
        participants: chat.participants.length,
        description: chat.description || ''
      };
    });

    res.json(details);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update allowlist (called by backend)
router.post('/allowlist', async (req, res) => {
  try {
    const { allowlist } = req.body;
    messageHandler.updateAllowlist(allowlist);
    res.json({ status: 'updated', count: allowlist.groups.length });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

---

### 6. Frontend - Group Selection UI

**File:** `frontend/components/WhatsAppGroupSelector.tsx`

```typescript
import { useState, useEffect } from 'react';

interface WhatsAppGroup {
  id: string;
  name: string;
  participants: number;
  description?: string;
}

const WhatsAppGroupSelector = () => {
  const [availableGroups, setAvailableGroups] = useState<WhatsAppGroup[]>([]);
  const [currentAllowlist, setCurrentAllowlist] = useState<WhatsAppGroup[]>([]);
  const [selectedIds, setSelectedIds] = useState<string[]>([]);
  const [loading, setLoading] = useState(true);
  const [saving, setSaving] = useState(false);

  useEffect(() => {
    loadData();
  }, []);

  const loadData = async () => {
    try {
      // Fetch all groups user is in
      const groupsRes = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/whatsapp/groups`);
      const groups = await groupsRes.json();
      setAvailableGroups(groups);

      // Fetch current allowlist
      const allowlistRes = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/whatsapp/allowlist`);
      const allowlist = await allowlistRes.json();
      setCurrentAllowlist(allowlist.groups || []);
      setSelectedIds((allowlist.groups || []).map(g => g.id));
    } catch (error) {
      console.error('Failed to load WhatsApp groups:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleSave = async () => {
    setSaving(true);
    try {
      await fetch(`${process.env.NEXT_PUBLIC_API_URL}/whatsapp/allowlist`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ group_ids: selectedIds })
      });
      alert('Allowlist updated successfully!');
      await loadData();  // Reload to show updated names
    } catch (error) {
      alert('Failed to update allowlist');
      console.error(error);
    } finally {
      setSaving(false);
    }
  };

  const toggleGroup = (groupId: string) => {
    setSelectedIds(prev =>
      prev.includes(groupId)
        ? prev.filter(id => id !== groupId)
        : [...prev, groupId]
    );
  };

  if (loading) return <div>Loading groups...</div>;

  return (
    <div className="whatsapp-group-selector">
      <h3>Select WhatsApp Groups to Monitor</h3>
      <p>Only selected groups will be summarized hourly (all private chats are always monitored)</p>

      <div className="group-list">
        {availableGroups.length === 0 ? (
          <p>No WhatsApp groups found. Make sure your WhatsApp is connected.</p>
        ) : (
          availableGroups.map(group => (
            <label key={group.id} className="group-item">
              <input
                type="checkbox"
                checked={selectedIds.includes(group.id)}
                onChange={() => toggleGroup(group.id)}
              />
              <div className="group-info">
                <div className="group-name">{group.name}</div>
                <div className="group-meta">
                  {group.participants} members
                  {group.description && ` • ${group.description.substring(0, 50)}`}
                </div>
                <div className="group-id">{group.id}</div>
              </div>
            </label>
          ))
        )}
      </div>

      <button onClick={handleSave} disabled={saving}>
        {saving ? 'Saving...' : 'Save Allowlist'}
      </button>

      <div className="current-allowlist">
        <h4>Currently Monitoring ({currentAllowlist.length} groups)</h4>
        <ul>
          {currentAllowlist.map(group => (
            <li key={group.id}>
              {group.name}
              {group.updated_at && (
                <span className="updated">
                  (Updated: {new Date(group.updated_at).toLocaleString()})
                </span>
              )}
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default WhatsAppGroupSelector;
```

---

## Testing

### Test Case 1: Group Name Change

```javascript
// Setup: Add group to allowlist
const groupId = "1234567890@g.us";
const initialName = "Project Team";

// 1. Add to allowlist
await updateAllowlist([groupId]);  // Name: "Project Team"

// 2. Send message - should be buffered
await sendMessageToGroup(groupId, "Test message 1");
assert(isBuffered(groupId) === true);

// 3. Rename group
await renameGroup(groupId, "CS101 Project Team");

// 4. Send another message - should STILL be buffered
await sendMessageToGroup(groupId, "Test message 2");
assert(isBuffered(groupId) === true);  // ✅ Still works!

// 5. Check database - name should be updated
const settings = await getUserSettings();
const group = settings.whatsapp_group_allowlist.groups.find(g => g.id === groupId);
assert(group.name === "CS101 Project Team");  // ✅ Name updated
```

### Test Case 2: Empty Allowlist

```javascript
// Setup: Empty allowlist
await updateAllowlist([]);

// Send messages to group
await sendMessageToGroup("1234567890@g.us", "Test");

// Should NOT be buffered
assert(isBuffered("1234567890@g.us") === false);

// Private chat should still work
await sendPrivateMessage("5555555555@c.us", "Test");
assert(isBuffered("5555555555@c.us") === true);  // ✅ Private chats always work
```

### Test Case 3: Group Not Found

```javascript
// Try to add non-existent group ID
const result = await updateAllowlist(["invalid-id@g.us"]);

// Should handle gracefully
assert(result.errors.length > 0);
assert(result.errors[0].includes("Group not found"));
```

---

## Migration Guide

### Migrating from Name-Based to ID-Based

If you previously used group names in the allowlist:

```python
# Migration script: migrate_allowlist.py

async def migrate_name_based_allowlist():
    """Convert old name-based allowlist to ID-based"""

    users = supabase.table('user_settings').select('*').execute()

    for user in users.data:
        old_allowlist = user.get('whatsapp_group_allowlist', [])

        # Old format: ["Group Name 1", "Group Name 2"]
        if isinstance(old_allowlist, list) and old_allowlist:
            print(f"Migrating user {user['user_id']}")

            # Query WhatsApp service for group IDs
            try:
                all_groups = await fetch_all_whatsapp_groups()

                new_groups = []
                for group_name in old_allowlist:
                    # Find group by name
                    match = next((g for g in all_groups if g['name'] == group_name), None)
                    if match:
                        new_groups.append({
                            "id": match['id'],
                            "name": match['name'],
                            "added_at": datetime.now().isoformat(),
                            "updated_at": datetime.now().isoformat()
                        })
                    else:
                        print(f"  ⚠️  Group not found: {group_name}")

                # Update to new format
                new_allowlist = {"groups": new_groups}
                supabase.table('user_settings').update({
                    'whatsapp_group_allowlist': new_allowlist
                }).eq('id', user['id']).execute()

                print(f"  ✅ Migrated {len(new_groups)} groups")

            except Exception as e:
                print(f"  ❌ Failed to migrate user: {e}")

if __name__ == '__main__':
    asyncio.run(migrate_name_based_allowlist())
```

---

## Benefits Summary

| Aspect | Name-Based | ID-Based |
|--------|-----------|----------|
| **Stability** | ❌ Breaks on rename | ✅ Never breaks |
| **Uniqueness** | ❌ Names can duplicate | ✅ Globally unique |
| **User Experience** | ❌ Must update allowlist after rename | ✅ Automatic name sync |
| **Implementation** | ✅ Simpler initially | ⚠️ Slightly more complex |
| **Debugging** | ❌ Hard to track renames | ✅ Clear audit trail |

---

## Summary

**Decision:** Use WhatsApp group ID (`chat.id._serialized`) for allowlist matching.

**Key Implementation Points:**
1. Store both ID and name in JSONB allowlist
2. Match by ID, display by name
3. Track group name changes and update database
4. Frontend shows human-readable names but submits IDs
5. Private chats always monitored (bypass allowlist)

**Result:** Robust, stable group identification that survives renames and provides good UX.
