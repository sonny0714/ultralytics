# Run Log — sonny

<!-- Long-running remote processes (started via `docker exec -d`) tracked here. -->
<!-- Lifecycle (3 stages):
       Active  : entry body lives in this file under "## Active"
       Done    : auto-detected finish → cut body to done/{run_id}.md, link under "## Done"
       Archive : user-confirmed cleanup → mv done/{run_id}.md to archive/{YYYY-MM-DD}/, summary under "## Archive"
                 (date = user cleanup date, not process finish date)
-->
<!--
Required fields per Active entry:
  - server, container, started, expected_duration, next_check_after
  - command, log_path, result_path
  - check_cmd, tail_cmd

Log path convention:
  {source_mnt_path}/tee_log/{project_name}/{topic}/{YYYYMMDD-HHMMSS}.log
  source_mnt_path = defaults.default_source_mnt_path (parent of all projects)
  → mounted, so docker storage does not grow.
-->

## Active

<!-- Example active entry (delete when first real entry is added) -->
<!--
### example_run_20260101-000000
- server: <server_name>
- container: <container_name>
- started: YYYY-MM-DD HH:MM:SS KST
- expected_duration: ~2h
- next_check_after: YYYY-MM-DD HH:MM:SS KST
- command: `python3 -u path/to/script.py --args`
- log_path: {source_mnt_path}/tee_log/{project_name}/{topic}/{YYYYMMDD-HHMMSS}.log
- result_path: path/to/output_dir/
- check_cmd: `ssh <server> "docker exec <container> pgrep -f script.py >/dev/null && echo running || echo done"`
- tail_cmd: `ssh <server> "docker exec <container> tail -20 {log_path}"`
-->

## Done

<!-- auto-completed runs awaiting user cleanup -->

## Archive

<!-- user-cleaned runs (link by YYYY-MM-DD cleanup date) -->
