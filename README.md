# SnowCraft Studio

SnowCraft Studio is a browser-based creative writing workspace. This README documents only behavior established by the supplied application code description. It does not treat placeholder controls or omitted code as implemented features.

> **Implementation note:** Some portions of the source were omitted and represented by comments such as `// ... existing code ...`. Features that cannot be verified from the available code are marked as incomplete or unverified rather than assumed to work.

## Features

| Area / tab | Status | What is implemented |
|---|---|---|
| Writing and formatting | **Working** | The available formatting controls can copy supported formats. |
| Voice narration | **Working** | Gemini is used for voice narration. Generated audio is kept only for the lifetime of the current browser tab/session; it is not persisted as a durable audio file. |
| AI Editor | **Working** | The AI Editor chat can send requests and display responses. Failed calls should be treated as errors; the current implementation may instead show a canned critique. |
| Vault / drafts | **Partially working; privacy issue** | Drafts are stored locally in the browser and in Firestore. Cloud drafts currently use a shared `public` path, which can expose one user's drafts to other signed-in users unless Firestore rules prevent access. |
| AI Studio actions | **Not implemented as AI features** | The controls named “Generate Creative Piece”, “Extend Next Lines”, “Vivid Sensory Polish”, “AI Art Prompt”, and “Social Hashtags” insert fixed text or show a message. They do not call Gemini. |
| Other controls or tabs | **Unverified / incomplete** | Source sections omitted from the supplied code cannot be documented as working. |

## Requirements

- A modern browser with JavaScript enabled.
- A Firebase project configured for the application.
- Firestore enabled if cloud draft storage is to be used.
- A Gemini API key for the Gemini-powered features.
- The host page must provide the application settings described below.

## Application settings

The application reads these values from the page environment:

| Setting | Purpose |
|---|---|
| `__firebase_config` | Firebase configuration used to initialize the Firebase client. |
| `__app_id` | Application identifier used by the app when organizing its data. |
| `__initial_auth_token` | Optional initial authentication token used to sign in when supplied. |

The exact provisioning mechanism depends on the host environment. Ensure these values are defined before the application initializes. Do not publish private credentials in client-side configuration.

## Setup

1. Create or select a Firebase project.
2. Enable the Firebase services required by the application, including Firestore and the authentication provider(s) used by the app.
3. Configure the host environment to provide `__firebase_config` and `__app_id`. Provide `__initial_auth_token` when the host environment uses it.
4. Configure Firestore security rules before allowing users to save cloud drafts.
5. Provide Gemini credentials through a secure server-side integration. **Do not expose a production Gemini API key in browser code or a browser request URL.**
6. Serve the application from its intended host environment and verify authentication, draft saving, formatting, narration, and AI Editor behavior.

This README does not prescribe a package-manager command because the available source description does not establish the repository's build tooling or scripts.

## Where data is stored

### Browser

Drafts are stored locally in the browser. Browser-local data is specific to the browser profile and device and may be cleared by the user or browser.

### Firestore

Drafts are also saved to Firestore. In the described implementation, cloud drafts are written under a shared `public` path and then merged into each user's Vault.

**This is a serious data-isolation problem.** A signed-in user—including an anonymously authenticated user—may be able to read or overwrite other users' drafts if the Firestore rules allow access to that path.

#### Recommended private-draft design

Store each user's drafts under a user-specific collection path, for example:

```text
users/{userId}/drafts/{draftId}
```

Use the authenticated Firebase user ID as `{userId}`. Query only the current user's draft collection, and validate ownership in Firestore rules. A conceptual rules pattern is:

```javascript
match /users/{userId}/drafts/{draftId} {
  allow read, create, update, delete: if
    request.auth != null && request.auth.uid == userId;
}
```

Adapt this pattern to the actual database structure and validate all fields and write operations required by the application. Do not rely on client-side filtering as a security boundary. Test with two separate accounts to confirm that neither account can read or modify the other's drafts.

### Generated audio

Generated narration audio exists only for the current browser tab/session. It is not retained as persistent cloud audio storage.

## Security and reliability issues

The following issues are present in, or follow directly from, the described implementation:

1. **Placeholder AI Studio buttons.** “Generate Creative Piece”, “Extend Next Lines”, “Vivid Sensory Polish”, “AI Art Prompt”, and “Social Hashtags” do not call Gemini. They insert fixed text or display a message.
2. **Shared cloud drafts.** Drafts are stored under a `public` Firestore path and merged into every user's Vault. Move drafts to per-user paths and enforce ownership in Firestore rules.
3. **Exposed Gemini API key.** The browser sends the key in the request URL. Browser users can inspect requests and obtain the key. Route Gemini requests through a trusted backend or serverless function, keep the key in server-side secrets, and apply appropriate authentication, validation, and rate limiting.
4. **Firestore write on every keystroke.** Saving each keystroke creates unnecessary writes, can increase cost, and may hit service limits. Debounce writes, save on meaningful intervals or events, and consider batching where appropriate.
5. **AI failures can look successful.** When an AI request fails, the app may display a canned critique instead of a clear error. Show an explicit failure state, preserve the user's input, and avoid presenting fallback text as a model-generated response.

## Known limitations / unfinished work

- Connect the five AI Studio placeholder actions to real Gemini requests, or label them clearly as unavailable.
- Move cloud drafts from the shared `public` path to per-user storage and deploy matching Firestore security rules.
- Remove the Gemini API key from browser-visible code and request URLs; use a secure server-side proxy.
- Debounce or otherwise reduce Firestore writes.
- Display clear errors when AI requests fail instead of canned responses.
- Review source sections omitted from the supplied code before documenting additional tabs or claiming additional features.
- Add tests for authentication, draft ownership, Firestore rules, AI request failures, and persistence behavior.

## Important deployment warning

Do not deploy the shared-draft implementation with permissive Firestore rules or a browser-exposed production Gemini key. Resolve the data-isolation and API-key issues before using the application with real users or private writing.
