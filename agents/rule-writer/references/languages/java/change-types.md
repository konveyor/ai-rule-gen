# Java Change Types

Use these categories when enumerating changes in Phase 1 (enumeration).

| Type | Description | Typically detectable? | Detection approach |
|---|---|---|---|
| `api_change` | Class, method, interface, or annotation renamed, removed, or relocated | Yes | `java.referenced` on the old FQN |
| `dependency_change` | Library added, removed, version constraint changed, scope changed | Yes | `java.dependency` on the artifact |
| `config_change` | Property key renamed, format changed, moved to different file | Yes | `builtin.filecontent` on the old key in `application.properties` / `application.yml` |
| `default_change` | Default value or behavior changed (e.g., timeout, pool size, retry policy) | Often | `builtin.filecontent` for explicit config of the old default, or `java.referenced` on the class whose default changed |
| `behavioral_change` | Runtime behavior differs, new constraints, auto-configuration removed | Sometimes | `java.referenced` on classes that depended on old behavior, or `builtin.filecontent` for config that assumed old behavior |
| `informational` | Context, background, no migration action needed | No | Not extracted — excluded from manifest |

## Detection hints for behavioral changes

When enumerating a `default_change` or `behavioral_change`, provide a `detection_hint` that answers: "Where would code relying on the OLD behavior be visible?"

Examples:
- "Default thread pool size changed from 200 to 10" -> hint: "Look for explicit `server.tomcat.threads.max` or code assuming high thread count"
- "Lazy initialization disabled by default" -> hint: "Look for `spring.main.lazy-initialization=true` or beans expecting lazy init"
- "Auto-configuration for DataSource removed when multiple beans present" -> hint: "Look for `@Autowired DataSource` without `@Primary`"
