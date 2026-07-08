# Apache Airflow Multi-Team bulk update authorization gap

## Status

Confirmed by Apache Airflow maintainers and fixed before release.

No CVE was issued because Multi-Team was an experimental feature and had not appeared in any released Apache Airflow version. The issue was fixed before Airflow 3.3.0 shipped.

## Summary

I reported a Multi-Team authorization gap in Apache Airflow's bulk update endpoints for Variables, Connections, and Pools.

In `[core] multi_team=True` mode, the single-resource update endpoints correctly checked both the existing resource team and the requested destination `team_name`.

However, the corresponding bulk update endpoints authorized only the resource's existing/source team. They did not authorize the destination `team_name` supplied in each bulk update request body before applying the update.

As a result, a user assigned only to `team_b` could use bulk update endpoints to move team-scoped resources from `team_b` into `team_a`, even though the equivalent single-resource update path correctly returned `403 Forbidden`.

## Affected area

- Project: Apache Airflow
- Feature area: Multi-Team authorization
- Auth manager used in PoC: SimpleAuthManager
- API area: public API bulk update endpoints
- Resource types:
  - Variables
  - Connections
  - Pools

## Affected endpoint classes

- `PATCH /api/v2/variables`
- `PATCH /api/v2/connections`
- `PATCH /api/v2/pools`

## Security boundary

A user belonging only to `team_b` should not be able to assign Variables, Connections, or Pools to `team_a`.

The single-resource update endpoints enforced this boundary correctly. The bulk update endpoints did not.

## Observed behavior

For each tested resource type, the single-resource update endpoint rejected the cross-team reassignment, while the bulk update endpoint accepted it and persisted the new `team_name`.

| Resource | Single PATCH `team_b -> team_a` | Bulk PATCH `team_b -> team_a` |
|---|---:|---:|
| Variables | `403 Forbidden` | `200 OK` |
| Connections | `403 Forbidden` | `200 OK` |
| Pools | `403 Forbidden` | `200 OK` |

After the bulk update, the moved resource became inaccessible to the original `team_b` user and accessible to a `team_a` user, confirming that the resource had crossed team boundaries.

## Impact

The issue broke Multi-Team resource isolation in the affected development line.

Depending on the resource type, this could allow cross-team resource injection or resource poisoning:

- Variables: injection of attacker-controlled runtime configuration into another team's Variable namespace.
- Connections: injection of attacker-controlled Connection metadata, endpoints, or credentials into another team's Connection namespace.
- Pools: injection or reassignment of scheduling-related Pool configuration into another team's namespace.

I did not claim remote code execution. The reported impact was cross-team resource injection / resource poisoning in Multi-Team mode.

## Vendor response

Apache Airflow maintainers confirmed that the reported cross-team bulk-update behavior was a real bug.

They stated that the issue would be fixed before Airflow 3.3.0 shipped, and that no CVE would be issued because Multi-Team was experimental and had not appeared in any released Airflow version.

## Fix direction

Bulk update authorization should check both:

1. the existing/source team of the resource from the database, and
2. the requested destination `team_name` from each bulk request entity.

This should match the behavior of the single-resource update endpoints.

## Credit

Reported by Trường Vũ Đức.
