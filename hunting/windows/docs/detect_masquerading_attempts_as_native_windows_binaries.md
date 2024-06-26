# Detect masquerading attempts as native Windows binaries

---

## Metadata

- **Author:** Elastic
- **UUID:** `93a72542-a1f7-4407-9175-8f066343db60`
- **Integration:** [endpoint](https://docs.elastic.co/integrations/endpoint)
- **Language:** `ES|QL`

## Query

```sql
from logs-endpoint.events.process-*
| where  @timestamp > NOW() - 7 day
| where event.type == "start" and event.action == "start" and host.os.name == "Windows" and not starts_with(process.executable, "C:\\Program Files\\WindowsApps\\") and not starts_with(process.executable, "C:\\Windows\\System32\\DriverStore\\") and process.name != "setup.exe"
| keep process.name.caseless, process.executable.caseless, process.code_signature.subject_name, process.code_signature.trusted, process.code_signature.exists, host.id
 /* system_bin contain Microsoft signed and located in system32 folder process names */
 /* non_system_bin contain non Microsoft signed process names */
| eval system_bin = case(starts_with(process.executable.caseless, "c:\\windows\\system32") and starts_with(process.code_signature.subject_name, "Microsoft") and process.code_signature.trusted == true, process.name.caseless, null), non_system_bin = case(process.code_signature.exists == false or process.code_signature.trusted != true or not starts_with(process.code_signature.subject_name, "Microsoft"), process.name.caseless, null)
 /* aggregate unique process name counts  by process.name and host.id */
| stats count_system_bin = count(system_bin), count_non_system_bin = count(non_system_bin) by process.name.caseless, host.id
 /* filter where the same process.name is present in both system_bin and non_system_bin */
| where count_system_bin >= 1 and count_non_system_bin >= 1
```

## Notes

- Output of the query is the process.name and host.id, you can pivot by host.id and process.name(non Microsoft signed) to find the specific suspicious instances.
- Potential false positives include processes with missing code signature details due to enrichment bugs.
- The queried index must capture process start events with code signature information (e.g. Windows event 4688 is not supported).
## MITRE ATT&CK Techniques

- [T1036](https://attack.mitre.org/techniques/T1036)

## License

- `Elastic License v2`
