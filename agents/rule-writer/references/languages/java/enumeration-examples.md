# Java Enumeration Examples

Worked examples showing how to enumerate changes from prose sections, especially behavioral/semantic changes that are easy to miss.

## Example 1: Default value change

**Guide text:**
> The default value for `server.servlet.session.timeout` has changed from 30 minutes to 15 minutes in Spring Boot 4.0. Applications relying on the previous 30-minute default should explicitly set the timeout.

**Enumeration:**
```json
{
  "description": "Default session timeout changed from 30 minutes to 15 minutes",
  "change_type": "default_change",
  "detectable": true,
  "detection_hint": "Look for applications NOT setting server.servlet.session.timeout explicitly — they silently get a shorter timeout. Also look for code that assumes 30-minute sessions.",
  "artifacts": ["server.servlet.session.timeout"]
}
```

**Reasoning:** The old default affects any application that didn't explicitly configure the timeout. A `builtin.filecontent` rule can detect the absence of explicit configuration, or warn when the property is present with the old value.

## Example 2: Auto-configuration removal

**Guide text:**
> Spring Boot 4.0 no longer auto-configures a `JdbcTemplate` bean when multiple `DataSource` beans are present. Applications must explicitly define which `DataSource` to use with `@Primary` or qualify the injection.

**Enumeration:**
```json
{
  "description": "JdbcTemplate auto-configuration removed when multiple DataSource beans present",
  "change_type": "behavioral_change",
  "detectable": true,
  "detection_hint": "Look for @Autowired JdbcTemplate or @Autowired DataSource without @Primary in applications that define multiple DataSource beans",
  "artifacts": ["JdbcTemplate", "DataSource"]
}
```

**Reasoning:** The change affects code that injects `JdbcTemplate` or `DataSource` without qualification. A `java.referenced` rule on `JdbcTemplate` with an appropriate message can warn users.

## Example 3: Deprecation to removal

**Guide text:**
> The `spring-boot-starter-data-ldap` starter has been removed. Use `spring-boot-starter-data-ldap3` instead.

**Enumeration:**
```json
{
  "description": "spring-boot-starter-data-ldap removed, replaced by spring-boot-starter-data-ldap3",
  "change_type": "dependency_change",
  "detectable": true,
  "detection_hint": "Look for spring-boot-starter-data-ldap in pom.xml or build.gradle",
  "artifacts": ["spring-boot-starter-data-ldap"]
}
```

**Reasoning:** "Removed" always means detectable — the old artifact is in the build file.

## Example 4: Not detectable (rare)

**Guide text:**
> The Spring team recommends reviewing your application's thread usage patterns before upgrading, as the virtual thread support may change how your application behaves under load.

**Enumeration:**
```json
{
  "description": "Virtual thread support may change application behavior under load",
  "change_type": "informational",
  "detectable": false,
  "detection_hint": null,
  "artifacts": []
}
```

**Reasoning:** This is general advice with no specific API, config key, or artifact to detect. No code footprint exists.
