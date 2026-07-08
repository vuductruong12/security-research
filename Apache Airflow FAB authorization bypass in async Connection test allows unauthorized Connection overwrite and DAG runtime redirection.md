Liên kết: [[Apache / Airflow # 68127](https://github.com/apache/airflow/pull/68127)]
# Apache Airflow FAB async Connection test authorization gap

## Status

Confirmed by Apache Airflow security maintainers and fixed before release.

No CVE was issued because the affected endpoint existed only in the unreleased 3.3.0 development line and the fix was merged before 3.3.0 shipped.

## Public reference

- Apache Airflow PR #68127: Require edit rights for async connection test updates

## Summary

I reported an authorization gap in Apache Airflow's FAB-protected asynchronous Connection test endpoint.

The issue allowed a low-privileged authenticated user with Connection read/create permissions, but without edit permission, to overwrite an existing persistent Connection by using the async Connection test flow with `commit_on_success=true`.

The normal Connection update path correctly rejected the same user with `403 Forbidden`, but the async test commit path reached an update operation through a create/test authorization path.

## Vendor response

Apache Airflow security maintainers confirmed that the reported behavior was a genuine authorization gap: a create-only user should not be able to overwrite an existing Connection through the async connection-test endpoint with `commit_on_success`.

They also confirmed that the fix was merged before release and that no CVE would be issued because the affected endpoint only existed in the unreleased 3.3.0 line.

## Affected area

- Project: Apache Airflow
- Component: FAB auth manager / public Connection API
- Endpoint area: async Connection test
- Security boundary: create/test permission vs edit/update permission

## Impact

The issue allowed unauthorized modification of existing Airflow Connection configuration in the affected development line.

Because Airflow Connections are resolved by DAGs at runtime, modifying an existing Connection can affect downstream workflow behavior. In my report, I demonstrated that a DAG using the same Connection ID could be redirected from a trusted local service to an attacker-controlled local service without modifying the DAG itself.

## Fix direction

The async Connection test commit path now requires edit authorization when it attempts to update an existing Connection.

## Credit

Reported by Trường Vũ Đức.
