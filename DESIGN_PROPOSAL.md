# Design Proposal: Claude Subscription Support

## Problem Statement

Users with Claude Pro/Team subscriptions who also have `ANTHROPIC_API_KEY` set in their environment variables inadvertently consume API credits when using VibeKanban with Claude Code agents. This happens because the API key takes precedence over subscription authentication, even when the user intends to use their subscription.

### Current Behavior

When launching a Claude Code agent:
1. VibeKanban spawns the agent process
2. The agent inherits the parent process's environment variables
3. If `ANTHROPIC_API_KEY` is present, Claude Code uses it automatically
4. User is charged API usage fees instead of using their subscription

### User Impact

- Unexpected API costs for subscription users
- Manual workaround required (unsetting environment variable before each launch)
- Poor user experience for users who use API keys for other projects

## Proposed Solution

Add a user-configurable setting in the Agents settings page that allows users to explicitly choose to use their Claude subscription when an API key is detected.

### UI Design

**Location:** Settings → Agents (`/settings/agents`)

**New Setting:**
```
☐ Use Claude subscription when API key is detected
  When enabled, VibeKanban will ignore ANTHROPIC_API_KEY and use your
  Claude Pro/Team subscription for Claude Code agents.
```

**Placement:** This setting should appear in the Claude Code agent configuration section, as it only affects Claude-based agents.

### Technical Design

#### 1. Data Model

Add a new boolean field to agent settings:

```typescript
interface AgentSettings {
  // ... existing fields
  useClaudeSubscription?: boolean; // defaults to false for backward compatibility
}
```

#### 2. Database Schema

Add migration to create new column:

```sql
ALTER TABLE agent_settings 
ADD COLUMN use_claude_subscription BOOLEAN DEFAULT FALSE;
```

#### 3. Frontend Implementation

**File:** `/frontend/src/pages/Settings/Agents.tsx` (approximate location)

```typescript
const [useSubscription, setUseSubscription] = useState(false);

<Checkbox
  id="use-claude-subscription"
  label="Use Claude subscription when API key is detected"
  checked={useSubscription}
  onChange={(checked) => {
    setUseSubscription(checked);
    saveAgentSettings({ useClaudeSubscription: checked });
  }}
  helpText="When enabled, VibeKanban will ignore ANTHROPIC_API_KEY and use your Claude Pro/Team subscription for Claude Code agents."
/>
```

#### 4. Backend Implementation

**File:** `/crates/*/agent_executor.rs` (approximate location)

```rust
// When spawning Claude Code agent process
fn spawn_claude_agent(settings: &AgentSettings) -> Result<Child> {
    let mut cmd = Command::new("npx");
    cmd.arg("@anthropics/claude-code");
    
    // Check if we should suppress the API key
    if settings.use_claude_subscription.unwrap_or(false) {
        // Remove ANTHROPIC_API_KEY from environment
        cmd.env_remove("ANTHROPIC_API_KEY");
    }
    
    cmd.spawn()
}
```

### User Flow

1. User navigates to Settings → Agents
2. User sees their `ANTHROPIC_API_KEY` is detected (optional: show indicator)
3. User enables "Use Claude subscription when API key is detected"
4. Setting is saved to database
5. When launching Claude Code agents, VibeKanban unsets the API key before spawning the process
6. Claude Code falls back to subscription authentication

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| No API key present | Setting has no effect (normal subscription flow) |
| API key present + setting disabled | Use API key (current behavior) |
| API key present + setting enabled | Unset API key, use subscription |
| User has no subscription | Claude Code will prompt for authentication |

### Backward Compatibility

- Default value is `false` (use API key if present)
- Existing users see no behavior change unless they opt in
- No breaking changes to existing workflows

## Alternative Solutions Considered

### 1. Environment Variable Override
**Approach:** Add `VIBE_USE_SUBSCRIPTION=true` environment variable

**Pros:**
- Simpler implementation
- No database changes needed

**Cons:**
- Still requires manual environment management
- Less discoverable than UI setting
- Doesn't solve the core UX problem

### 2. Startup Prompt
**Approach:** Prompt user at startup when API key is detected

**Pros:**
- Very explicit
- No persistent storage needed

**Cons:**
- Interrupts workflow on every launch
- Annoying for users who want consistent behavior
- Not suitable for automated/headless usage

### 3. Per-Task Override
**Approach:** Add checkbox to each task creation dialog

**Pros:**
- Maximum flexibility per task

**Cons:**
- Decision fatigue (every task)
- More complex UI
- Harder to set consistent default

**Why the proposed solution is better:** Settings page configuration provides the right balance of discoverability, user control, and convenience without disrupting workflow.

## Success Metrics

- Users with subscriptions stop reporting unexpected API charges
- No increase in authentication-related support issues
- Setting is adopted by users who have both API keys and subscriptions

## Testing Plan

### Unit Tests
- Test agent executor with setting enabled/disabled
- Test environment variable removal logic
- Test database migration

### Integration Tests
- Spawn agent with API key present + setting enabled → verify no API usage
- Spawn agent with API key present + setting disabled → verify API usage
- Spawn agent with no API key → verify normal subscription flow

### Manual Testing
- Test on macOS, Linux, Windows
- Test with Claude Pro subscription
- Test with Claude Team subscription
- Test with invalid/expired API key + subscription fallback

## Implementation Checklist

- [ ] Database migration for new setting
- [ ] Frontend: Add checkbox to settings page
- [ ] Frontend: API calls to save/load setting
- [ ] Backend: Update settings schema
- [ ] Backend: Modify agent executor to respect setting
- [ ] Unit tests for backend logic
- [ ] Integration tests for agent spawning
- [ ] Update documentation (if needed)
- [ ] Test on all supported platforms

## Timeline Estimate

- Design review: 1-2 days
- Implementation: 4-8 hours
- Testing: 2-4 hours
- Code review iterations: 1-2 days

**Total: ~1 week** from approval to merge

## Open Questions

1. Should we show an indicator when an API key is detected on the settings page?
2. Should this setting apply globally or per-project?
3. Should we log when the API key is being suppressed for debugging?
4. Do other agents (Cursor, Codex) have similar subscription/API key conflicts?

## References

- Working proof-of-concept: https://github.com/potable-anarchy/vibe-kanban-launcher
- Related discussion: [TBD - link to GitHub discussion once created]
