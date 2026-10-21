# Building a QQ AI Bot on Windows: From First Connection to Migration

English | [简体中文](windows-qq-ai-bot.zh-CN.md)

These notes summarize one local deployment, recorded through October 2026. Account identifiers, personal nicknames, device usernames, actual local paths, and private configuration are omitted. Settings describe that deployment and are not defaults or recommendations for other versions.

## 1. Understand the pipeline and test each layer

The deployment used this pipeline:

```text
QQ → Adapter runtime (choose NapCat or SnowLuma)
   → OneBot v11 reverse WebSocket → AstrBot → Model API
```

QQ provides account login and messaging. The adapter runtime translates messages. AstrBot manages platform adapters, models, personas, and plugins. The model service generates responses.

Test each layer separately. A connected WebSocket confirms only the protocol connection. A command response confirms message transport. A normal chat response also checks model invocation. Plugins, images, and group permissions need separate tests.

## 2. Keep the materials needed to reproduce the setup

Recorded versions were AstrBot 4.29.1, Python 3.12, NapCat 4.18.38, SnowLuma 2.0.3, and Private Companion 6.5.6. This version record does not guarantee compatibility with other QQ builds or future releases.

A useful setup order is:

1. Create a dedicated project directory with separate locations for installers, programs, runtime data, and scripts.
2. Prepare Python, the AstrBot source, and a matching dashboard release.
3. Install dependencies and retain a dependency version snapshot and the actual installation packages.
4. Choose one QQ adapter runtime and establish account login and the OneBot connection.
5. Configure the model service and verify both commands and ordinary chat.
6. Add personas, companion features, split replies, and the reaction image library afterward.

A version string does not necessarily identify the exact source contents. A main-branch snapshot downloaded on a particular day may differ from a release tag with the same displayed version. Keep the actual source archive, commit information, and dependency snapshot when reproducibility matters.

A locally recorded SHA256 helps detect changes during later copying. It does not replace a publisher signature or verification of the download source.

## 3. Key OneBot settings

These examples assume both services run on the same computer. Check every value in your environment.

| Setting | Example or requirement |
|---|---|
| Connection direction | The adapter runtime connects to AstrBot as a WebSocket client |
| Reverse WebSocket URL | `ws://127.0.0.1:6199/ws` |
| Message format | Array |
| Communication token | Use the same self-generated secret at both ends |
| Heartbeat / reconnect interval | 5000 ms in this deployment |
| AstrBot adapter | OneBot v11, using the `aiocqhttp` type in this deployment |

Dashboard credentials, the OneBot communication token, and the model API key are separate credentials and are not interchangeable.

Dashboard ports in this setup were 6185 for AstrBot, 6099 for NapCat, and 5099 for SnowLuma. Access methods and authentication depend on the installed version. Loopback addresses work for a single computer; deployments across devices or containers require a fresh review of addresses, authentication, and network exposure.

Administrator accounts and session allowlists also serve different purposes. Administrator status does not itself authorize a session. Private-chat rules should use the complete session ID supplied by the installed version. In the recorded version, an empty allowlist imposed no restriction. Test both allowed and disallowed sessions after configuration.

## 4. Troubleshoot by symptom

| Symptom | Checks informed by this deployment |
|---|---|
| WebSocket connection refused | Check that AstrBot is running, confirm its listening address and port, and remove stale connection URLs |
| WebSocket returns 403 | First compare the communication tokens at both ends |
| `/help` works but normal chat does not | Check the global AI switch, selected model, and session permissions, then inspect model invocation logs |
| Model listing or invocation reports a connection error | Check network access in the launch environment, then credentials and the service URL; this error alone does not prove the key is invalid |
| Persona behavior differs from expectations | Check the current session persona, previous context, and dynamic plugin injections |
| A plugin is enabled but has no effect | Check the target user, authorization, assets, required services, and trigger conditions |
| Dashboard is online but messages do not arrive | Send a test message and compare actual event timestamps in the adapter runtime and AstrBot |

Change one layer at a time and run the corresponding test so that the effect remains traceable.

## 5. Verify independence when switching adapter runtimes

During the switch from NapCat to SnowLuma, SnowLuma initially attached to a QQ instance launched by NapCat. Stopping NapCat also closed that QQ instance, breaking the new route.

The solution was to stop the original runtime and its QQ instance, log the bot account into the normal desktop QQ client, and manually select that process in SnowLuma. Avoid selecting another personal QQ process.

Switching procedure:

1. Stop the old runtime and its bot QQ instance.
2. Log the bot account into the normal desktop QQ client.
3. Start the new runtime, complete its required setup through its interface, and manually choose the target process.
4. Check the OneBot URL, token, role, and message format. Do not copy configuration JSON directly from a different runtime.
5. Keep the existing AstrBot platform ID where possible, and review sessions, allowlists, and plugin data.
6. Test commands, chat, and images while the old runtime remains stopped.

Do not let two runtimes handle the same bot account simultaneously. To roll back, stop the new runtime, exit its QQ instance, restart through the old route, and retest.

Independent messaging through SnowLuma and normal desktop QQ was confirmed in this deployment. There was insufficient continuous runtime evidence to establish greater stability than the other route. Long-term stability, GIFs, and group permissions need separate validation.

## 6. Configure personas, companion state, and long-term memory separately

Chat style was influenced by persona text, previous conversation context, and plugin features such as daily state, emotional state, and expression learning.

First verify the base persona, then enable dynamic features individually. Some passive state injections and expression learning were paused to stabilize the style in this deployment. That was a specific debugging choice, not a universal best configuration.

Other lessons:

- Saving a persona Markdown file does not update the runtime persona automatically; import or edit it through the dashboard.
- Starting a new conversation can remove the influence of old context, but does not prove that long-term memory has been deleted.
- Companion profiles and relationship data are distinct from a complete long-term memory system.
- Installation of the long-term memory core and embedding service was not completed here, so those capabilities were not verified.

## 7. Validate a small reaction image library first

Copying images into a directory does not automatically make them searchable by the plugin. Use a supported import method to create metadata and indexes.

Start with a small set of images whose sources and usage rights are clear. Record a name, tags, emotions, intended use, description, and visible text for each image. Leave absent text blank and do not invent OCR results. Distinguish the image's emotion from its purpose: comforting a user does not necessarily call for a crying image.

Test explicit image requests before automatic selection in ordinary conversation. Trigger probability, cooldowns, repetition limits, and special semantic rules can all affect the result. Several consecutive messages without an image do not necessarily indicate a failure.

Later records confirmed a small library imported through plugin methods and one successful contextual match and image send on the NapCat route. The earlier plan saying assets had not been imported was outdated. Automatic images and GIFs on SnowLuma still required separate tests. Tune probability and cooldown values for your own usage.

Keyword and tag matching can establish the basic workflow. Configure a separate embedding service only after confirming a need for semantic retrieval. A chat model cannot simply substitute for an embedding model.

A compatibility issue also caused the model to emit a tool call as ordinary text. A local patch addressed that scenario for the installed version. Recheck that patch and command aliases after core or plugin updates.

## 8. Use different materials for migration and public sharing

Personal migration requires complete runtime data. Public sharing should contain anonymized explanations and purpose-built example configurations.

Stop services normally before backing up the entire data directory. With SQLite WAL enabled, copying only a running database's `.db` file may omit data. Copy the complete directory after shutdown, or use SQLite's online backup interface.

A Python virtual environment copied to another path may not work. Rebuild it on the new computer using the recorded Python version and dependency snapshot, then check absolute paths in configuration and image libraries. QQ login state and process IDs are not reliable portable setup materials; log in again and select the process on the new machine.

Exclude the following from public materials:

- API keys, communication tokens, dashboard passwords, login QR codes, and session credentials.
- Real account identifiers, personal nicknames, contacts, group information, and device paths.
- Chat databases, user profiles, relationship records, diaries, memory data, and private image metadata.
- Raw logs, screenshots, old configuration backups, crash dumps, and unreviewed scripts or archives.

Old backups may retain previous keys and conversations. Git ignore rules reduce accidental commits but do not establish that a repository, its history, or a manually assembled archive is anonymized. Share this standalone guide without attaching the full project directory.

## 9. Minimum acceptance checks

| Test | Passing condition |
|---|---|
| Commands | A test account receives a normal response to `/help` |
| Model | Ordinary questions receive replies without corresponding model invocation errors |
| Access control | Allowed sessions work and disallowed sessions are correctly restricted |
| Persona | New conversations use the intended persona and plugin effects are understood |
| Image library | Explicit requests work, and automatic selection follows context and cooldown rules |
| Restart | Configuration, assets, and plugin data remain usable |
| Runtime switch | The new route still sends and receives while the old runtime is stopped |
| Stability | Observe continuously and record disconnections, QQ status, and recovery steps |

One successful text reply proves only that the tested path worked at that moment. Keep separate results for another-machine migration, sustained operation, images, and group features.

## References

These are project entry points cited in the original deployment notes. Follow the documentation for your installed version. Links and latest releases were not reverified for this guide.

- [AstrBot](https://github.com/AstrBotDevs/AstrBot)
- [NapCat Windows Shell setup](https://napneko.github.io/guide/boot/Shell)
- [SnowLuma](https://github.com/SnowLuma/SnowLuma)
- [Private Companion](https://github.com/menglimi/astrbot_plugin_private_companion)
- [uv Python environment management](https://docs.astral.sh/uv/guides/install-python/)
- [DeepSeek API documentation](https://api-docs.deepseek.com/)
