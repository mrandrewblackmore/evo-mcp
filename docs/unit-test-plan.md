# Comprehensive Unit Test Plan for `evo-mcp`

> Generated: March 5, 2026
> Status: Planning — no tests implemented yet

## Testing Infrastructure

- **Framework**: `pytest` + `pytest-asyncio` (already in dev dependencies)
- **Proposed structure**: `tests/` directory mirroring `src/` layout
- **Key approach**: Mock all Evo SDK clients (`WorkspaceAPIClient`, `ObjectAPIClient`, `DiscoveryAPIClient`, `AioTransport`) and filesystem I/O where needed

---

## 1. `src/evo_mcp/__init__.py` — Package Version Detection

| # | Function/Path | Test Description | Category |
|---|---|---|---|
| 1.1 | `__version__` via `importlib.metadata` | Verify version string loads from installed package metadata | Pure logic |
| 1.2 | `__version__` fallback to `pyproject.toml` | Mock `PackageNotFoundError`, verify TOML parsing fallback | Pure logic (mock) |
| 1.3 | `__version__` double fallback | Mock both metadata and TOML failure → verify `"unknown"` | Pure logic (mock) |

**Test file**: `tests/test_init.py`

---

## 2. `src/evo_mcp/context.py` — EvoContext Class

| # | Method | Test Description | Category |
|---|---|---|---|
| 2.1 | `__init__()` | Verify default state: `_initialized=False`, `org_id=None`, `hub_url=None`, cache_path created | Pure logic |
| 2.2 | `load_variables_from_cache()` | Write a valid `variables.json`, load it, verify `org_id` (UUID) and `hub_url` are set | Filesystem mock |
| 2.3 | `load_variables_from_cache()` — missing file | Verify graceful no-op when `variables.json` doesn't exist | Filesystem mock |
| 2.4 | `save_variables_to_cache()` | Set `org_id` and `hub_url`, call save, read file back and verify JSON content + UUID→str conversion | Filesystem mock |
| 2.5 | `get_access_token_from_cache()` — valid token | Write a non-expired JWT to cache, verify it's returned | Pure logic (mock fs + jwt) |
| 2.6 | `get_access_token_from_cache()` — expired token | Write an expired JWT, verify `None` returned | Pure logic (mock jwt) |
| 2.7 | `get_access_token_from_cache()` — no cache file | Verify `None` when cache file doesn't exist | Pure logic |
| 2.8 | `save_access_token_to_cache()` | Save a token string, read back JSON file, verify `{'access_token': ...}` | Filesystem mock |
| 2.9 | `get_transport()` | Verify returns `AioTransport` singleton (second call returns same object) | Mock SDK |
| 2.10 | `get_access_token_via_user_login()` | Mock `OAuthConnector`, `AuthorizationCodeAuthorizer.login()`, `get_default_headers()` → verify token extracted from `Bearer` header | Async + Mock SDK |
| 2.11 | `get_access_token_via_user_login()` — missing CLIENT_ID | Verify `ValueError` raised when `EVO_CLIENT_ID` not set | Async + env mock |
| 2.12 | `get_authorizer()` — cached token | Mock `get_access_token_from_cache()` → returns token, verify no OAuth flow triggered | Async + mock |
| 2.13 | `get_authorizer()` — no cached token | Mock cache miss, mock OAuth flow, verify `save_access_token_to_cache` called | Async + mock |
| 2.14 | `initialize()` — first call | Mock authorizer + discovery client → verify `workspace_client` created, `_initialized=True` | Async + mock SDK |
| 2.15 | `initialize()` — already initialized | Call twice, verify second call is no-op (short-circuits) | Async + mock |
| 2.16 | `initialize()` — no organizations | Mock empty `list_organizations()` → verify `ValueError` | Async + mock |
| 2.17 | `initialize()` — no hubs | Mock org with empty hubs → verify `ValueError` | Async + mock |
| 2.18 | `get_object_client()` | Mock `workspace_client.get_workspace()` → verify `ObjectAPIClient` created with correct environment | Async + mock |
| 2.19 | `switch_instance()` | Call with new org_id/hub_url → verify `connector`, `workspace_client` recreated, `save_variables_to_cache` called | Async + mock |
| 2.20 | `ensure_initialized()` (module-level) | Verify it calls `evo_context.initialize()` | Async + mock |

**Test file**: `tests/test_context.py`

---

## 3. `src/evo_mcp/tools/general_tools.py` — 7 General Tools

| # | Tool Function | Test Description | Category |
|---|---|---|---|
| 3.1 | `workspace_health_check()` — with ID | Mock `workspace_client.get_workspace()`, verify returned dict has workspace name + status | Async + mock |
| 3.2 | `workspace_health_check()` — no ID | Verify message about no `workspace_id` | Async + mock |
| 3.3 | `list_workspaces()` — defaults | Mock `list_workspaces()`, verify result list formatting | Async + mock |
| 3.4 | `list_workspaces()` — with name filter | Verify name filter applied in SDK call | Async + mock |
| 3.5 | `list_workspaces()` — with `include_deleted=True` | Verify deleted filter passed | Async + mock |
| 3.6 | `get_workspace()` — by ID (UUID) | Mock get_workspace, verify workspace dict returned | Async + mock |
| 3.7 | `get_workspace()` — by name | Mock list_workspaces with name match → verify resolution to ID | Async + mock |
| 3.8 | `get_workspace()` — not found | Mock empty results → verify error message | Async + mock |
| 3.9 | `list_objects()` — defaults | Mock object client list_objects → verify formatting | Async + mock |
| 3.10 | `list_objects()` — with schema_id filter | Verify filter passed to SDK | Async + mock |
| 3.11 | `list_objects()` — with `include_deleted` | Verify parameter forwarded | Async + mock |
| 3.12 | `get_object()` — by ID | Mock get_object → verify returned dict | Async + mock |
| 3.13 | `get_object()` — by path | Mock get_object with path → verify resolution | Async + mock |
| 3.14 | `get_object()` — with version | Verify version parameter passed | Async + mock |
| 3.15 | `list_my_instances()` | Mock `discovery_client.list_organizations()` → verify formatting | Async + mock |
| 3.16 | `select_instance()` — by name | Mock org list, match by name → verify `switch_instance()` called | Async + mock |
| 3.17 | `select_instance()` — by ID | Mock org list, match by ID → verify switch | Async + mock |
| 3.18 | `select_instance()` — not found | Verify error message returned | Async + mock |

**Test file**: `tests/tools/test_general_tools.py`

---

## 4. `src/evo_mcp/tools/admin_tools.py` — 5 Admin Tools

| # | Tool Function | Test Description | Category |
|---|---|---|---|
| 4.1 | `create_workspace()` | Mock `workspace_client.create_workspace()` → verify returned workspace dict | Async + mock |
| 4.2 | `get_workspace_summary()` | Mock list_objects → verify object count-by-schema aggregation logic | Async + mock |
| 4.3 | `create_workspace_snapshot()` — without data | Mock `create_snapshot` → verify snapshot ID returned | Async + mock |
| 4.4 | `create_workspace_snapshot()` — with data blob refs | Verify data refs extracted and passed | Async + mock |
| 4.5 | `workspace_copy_object()` | Mock source+dest clients, `copy_object_data()` → verify object created in destination | Async + mock |
| 4.6 | `workspace_copy_object()` — with data blobs | Verify `copy_object_data` called for each blob ref | Async + mock |
| 4.7 | `workspace_duplicate_workspace()` — basic | Mock full workflow (create workspace → list objects → copy each) | Async + mock |
| 4.8 | `workspace_duplicate_workspace()` — with schema filter | Verify only matching schemas copied | Async + mock |
| 4.9 | `workspace_duplicate_workspace()` — with name filter | Verify name pattern filter applied | Async + mock |

**Test file**: `tests/tools/test_admin_tools.py`

---

## 5. `src/evo_mcp/tools/data_tools.py` — 4 Data Tools (COMMENTED OUT)

| # | Tool Function | Test Description | Category |
|---|---|---|---|
| 5.1 | `create_object()` | Mock `object_client.create_object()` → verify returned ID | Async + mock |
| 5.2 | `get_object_content()` | Mock `get_object` + JSON download → verify full content returned | Async + mock |
| 5.3 | `get_object_versions()` | Mock version listing → verify formatted version list | Async + mock |
| 5.4 | `extract_data_references()` | Mock object content → verify delegation to `evo_data_utils.extract_data_references()` | Async + mock |

**Test file**: `tests/tools/test_data_tools.py`

---

## 6. `src/evo_mcp/tools/filesystem_tools.py` — 3 Tools + 1 Helper

| # | Function | Test Description | Category |
|---|---|---|---|
| 6.1 | `_get_data_directory()` — env var set | Set `EVO_LOCAL_DATA_DIR` env → verify returns that path | Pure logic (env mock) |
| 6.2 | `_get_data_directory()` — env not set | Unset env → verify fallback to `~/evo_local_data` | Pure logic (env mock) |
| 6.3 | `configure_local_data_directory()` — valid path | Mock `os.path.isdir()`, verify directory set in global | Async + fs mock |
| 6.4 | `configure_local_data_directory()` — `create_if_missing=True` | Verify `os.makedirs` called | Async + fs mock |
| 6.5 | `configure_local_data_directory()` — invalid path, no create | Verify error message | Async + fs mock |
| 6.6 | `list_local_data_files()` — flat listing | Create tmp dir with files → verify glob results | Async + tmp dir |
| 6.7 | `list_local_data_files()` — with pattern | Verify `*.csv` pattern filters correctly | Async + tmp dir |
| 6.8 | `list_local_data_files()` — recursive | Verify nested files found with `recursive=True` | Async + tmp dir |
| 6.9 | `preview_csv_file()` — valid CSV | Create a tmp CSV → verify column names, types, row count, stats | Async + tmp file |
| 6.10 | `preview_csv_file()` — file not found | Verify error message | Async + fs mock |
| 6.11 | `preview_csv_file()` — non-CSV file | Verify error or graceful handling | Async + fs mock |

**Test file**: `tests/tools/test_filesystem_tools.py`

---

## 7. `src/evo_mcp/tools/object_build_tools.py` — 4 Builder Tools

| # | Tool Function | Test Description | Category |
|---|---|---|---|
| 7.1 | `build_and_create_pointset()` — dry_run valid CSV | Provide valid CSV with X,Y,Z + attributes → verify schema validation passes, no SDK call | Async + mock |
| 7.2 | `build_and_create_pointset()` — dry_run missing columns | CSV without required X/Y/Z → verify validation error | Async + mock |
| 7.3 | `build_and_create_pointset()` — actual create | Mock `object_client.create_object()` → verify object created | Async + mock |
| 7.4 | `build_and_create_pointset()` — file not found | Invalid CSV path → verify error message | Async + mock |
| 7.5 | `build_and_create_line_segments()` — dry_run valid | Two valid CSVs (vertices + segments) → verify validation | Async + mock |
| 7.6 | `build_and_create_line_segments()` — invalid segment refs | Segment indices out of range → verify error | Async + mock |
| 7.7 | `build_and_create_line_segments()` — actual create | Mock SDK → verify creation | Async + mock |
| 7.8 | `build_and_create_downhole_collection()` — dry_run valid | Collar + survey + intervals CSVs → verify full build | Async + mock |
| 7.9 | `build_and_create_downhole_collection()` — missing collar columns | Verify validation error for missing required columns | Async + mock |
| 7.10 | `build_and_create_downhole_collection()` — mismatched hole IDs | IDs in intervals not in collar → verify error/warning | Async + mock |
| 7.11 | `build_and_create_downhole_intervals()` — dry_run valid | Valid intervals CSV → verify validation | Async + mock |
| 7.12 | `build_and_create_downhole_intervals()` — missing from/to columns | Verify validation error | Async + mock |

**Test file**: `tests/tools/test_object_build_tools.py`

---

## 8. `src/evo_mcp/tools/instance_users_admin_tools.py` — 5 Tools + 2 Helpers

| # | Function | Test Description | Category |
|---|---|---|---|
| 8.1 | `get_workspace_client()` (helper) | Verify returns `evo_context.workspace_client` after `ensure_initialized()` | Async + mock |
| 8.2 | `read_pages_from_api()` — single page | Mock API returning < limit items → verify single call, all items returned | Async + mock |
| 8.3 | `read_pages_from_api()` — multiple pages | Mock API returning full pages then partial → verify pagination loop | Async + mock |
| 8.4 | `read_pages_from_api()` — with `up_to` limit | Verify results truncated at `up_to` count | Async + mock |
| 8.5 | `get_users_in_instance()` | Mock paginated user list → verify formatted output | Async + mock |
| 8.6 | `list_roles_in_instance()` | Mock `list_roles` → verify role list returned | Async + mock |
| 8.7 | `add_users_to_instance()` — single user | Mock `add_users` → verify user added with role | Async + mock |
| 8.8 | `add_users_to_instance()` — batch | Mock with multiple email+role pairs → verify batch call | Async + mock |
| 8.9 | `remove_user_from_instance()` | Mock `remove_user` → verify deletion | Async + mock |
| 8.10 | `update_user_role_in_instance()` | Mock `update_user_role` → verify role change | Async + mock |

**Test file**: `tests/tools/test_instance_users_admin_tools.py`

---

## 9. `src/evo_mcp/utils/object_builders.py` — BaseObjectBuilder + 4 Subclasses

This is the **highest-value test target** — ~1021 lines of pure data transformation logic.

### 9a. BaseObjectBuilder Methods

| # | Method | Test Description | Category |
|---|---|---|---|
| 9.1 | `save_lookup_table()` | Provide a DataFrame → mock `data_client.save_data()` → verify `LookupTable_V1_0_1` dict structure | Mock data_client |
| 9.2 | `save_int_array()` | Provide integer Series → verify `IntegerArray1_V1_0_1` schema output | Mock data_client |
| 9.3 | `save_float_array1()` | Provide float Series → verify `FloatArray1_V1_0_1` schema output | Mock data_client |
| 9.4 | `save_float_array2()` | Provide DataFrame with 2 float columns → verify `FloatArray2_V1_0_1` | Mock data_client |
| 9.5 | `save_float_array3()` | Provide DataFrame with 3 float cols (X,Y,Z) → verify `FloatArray3_V1_0_1` | Mock data_client |
| 9.6 | `save_index_array2()` | Provide DataFrame with 2 int columns → verify schema output | Mock data_client |
| 9.7 | `build_category_attribute()` — basic | Series with string categories → verify `CategoryAttribute` with correct lookup table + indices | Mock data_client |
| 9.8 | `build_category_attribute()` — with NaN | Series containing NaN → verify `NanCategorical` handling | Mock data_client |
| 9.9 | `build_continuous_attribute()` — basic | Float Series → verify `ContinuousAttribute` with values array | Mock data_client |
| 9.10 | `build_continuous_attribute()` — with NaN | Series with NaN → verify `NanContinuous` handling | Mock data_client |
| 9.11 | `build_attribute()` — numeric column | Auto-detect as continuous → verify dispatches to `build_continuous_attribute` | Mock data_client |
| 9.12 | `build_attribute()` — string column | Auto-detect as categorical → verify dispatches to `build_category_attribute` | Mock data_client |
| 9.13 | `build_attributes()` — multiple columns | DataFrame with mixed types → verify list of attributes | Mock data_client |
| 9.14 | `build_attributes()` — exclude columns | Verify excluded columns are skipped | Mock data_client |
| 9.15 | `build_bounding_box()` — 3D data | DataFrame with X,Y,Z → verify min/max coordinates in `BoundingBox_V1_0_1` | **Pure logic** |
| 9.16 | `build_bounding_box()` — with NaN coordinates | Verify NaN values excluded from min/max | **Pure logic** |
| 9.17 | `validate_object()` — valid dict | Provide schema-compliant dict → verify no errors | Pure logic |
| 9.18 | `validate_object()` — invalid dict | Provide non-compliant dict → verify `ValidationError` | Pure logic |
| 9.19 | `_add_error()` / `_add_warning()` | Verify messages appended to `self.errors` / `self.warnings` | **Pure logic** |
| 9.20 | `reset_messages()` | Verify both lists cleared | **Pure logic** |

### 9b. PointsetBuilder

| # | Method | Test Description | Category |
|---|---|---|---|
| 9.21 | `build()` — valid data | DataFrame with X,Y,Z + attributes → verify complete `Pointset_V1_3_0` dict | Mock data_client |
| 9.22 | `build()` — minimal (X,Y,Z only) | No extra attributes → verify valid pointset with just locations | Mock data_client |
| 9.23 | `build()` — large dataset | Performance test with 10k+ rows | Mock data_client |

### 9c. LineSegmentsBuilder

| # | Method | Test Description | Category |
|---|---|---|---|
| 9.24 | `build()` — valid data | Vertices DF + segments DF → verify `LineSegments_V2_2_0` dict | Mock data_client |
| 9.25 | `build()` — with vertex attributes | Verify attributes attached to vertices | Mock data_client |
| 9.26 | `build()` — with segment attributes | Verify attributes attached to segments | Mock data_client |

### 9d. DownholeCollectionBuilder

| # | Method | Test Description | Category |
|---|---|---|---|
| 9.27 | `build_hole_id_lookup()` | List of hole IDs → verify lookup DataFrame structure | **Pure logic** |
| 9.28 | `save_path_array()` | DataFrame with depth/azimuth/dip → verify parquet save | Mock data_client |
| 9.29 | `build_hole_index_map()` — all IDs match | Verify mapping DataFrame with correct indices | **Pure logic** |
| 9.30 | `build_hole_index_map()` — unmatched IDs | Data IDs not in lookup → verify error/warning logged | **Pure logic** |
| 9.31 | `save_holes_mapping()` | Holes DataFrame → verify parquet save structure | Mock data_client |
| 9.32 | `build_location()` — valid collar+survey | Complete collar and survey DataFrames → verify Location dict | Mock data_client |
| 9.33 | `build_location()` — missing survey data | Collar only → verify handling | Mock data_client |
| 9.34 | `build_interval_collection()` — valid | Intervals DataFrame with from/to + attributes → verify collection dict | Mock data_client |
| 9.35 | `build_interval_collection()` — multiple interval tables | Multiple CSVs → verify each interval table built | Mock data_client |
| 9.36 | `build()` — full integration | All 3 CSVs (collar+survey+intervals) → verify complete `DownholeCollection_V1_3_0` | Mock data_client |
| 9.37 | `build()` — multi-hole collection | 5+ holes with multiple interval tables → verify all correctly mapped | Mock data_client |

### 9e. DownholeIntervalsBuilder

| # | Method | Test Description | Category |
|---|---|---|---|
| 9.38 | `build()` — valid data | Intervals DataFrame with from/to + attributes → verify `DownholeIntervals_V1_3_0` | Mock data_client |
| 9.39 | `build()` — multiple attributes | Mix of categorical + continuous → verify all built | Mock data_client |

**Test file**: `tests/utils/test_object_builders.py`

---

## 10. `src/evo_mcp/utils/evo_data_utils.py` — 2 Utility Functions

| # | Function | Test Description | Category |
|---|---|---|---|
| 10.1 | `extract_data_references()` — flat dict | Dict with `$data_ref` keys → verify all refs found | **Pure logic** |
| 10.2 | `extract_data_references()` — nested dict | Deeply nested dicts → verify recursive traversal | **Pure logic** |
| 10.3 | `extract_data_references()` — lists containing dicts | Verify traversal into list elements | **Pure logic** |
| 10.4 | `extract_data_references()` — no refs | Dict without any `$data_ref` → verify empty list | **Pure logic** |
| 10.5 | `extract_data_references()` — empty dict | `{}` → verify empty list | **Pure logic** |
| 10.6 | `copy_object_data()` | Mock source + dest data clients → verify each blob ref copied | Async + mock |

**Test file**: `tests/utils/test_evo_data_utils.py`

---

## 11. `src/mcp_tools.py` — Entry Point / Server Config

| # | Function/Logic | Test Description | Category |
|---|---|---|---|
| 11.1 | `_get_objects_reference_content()` | Verify OBJECTS.md loaded as string | Filesystem |
| 11.2 | `_get_objects_reference_content()` — missing file | Mock missing file → verify graceful handling | Filesystem mock |
| 11.3 | `get_objects_reference()` | Verify returns content string (MCP resource) | Pure logic |
| 11.4 | Tool registration — `MCP_TOOL_FILTER="all"` | Verify all tool registration functions called | Config + mock |
| 11.5 | Tool registration — `MCP_TOOL_FILTER="admin"` | Verify only admin + general + instance_users tools registered | Config + mock |
| 11.6 | Tool registration — `MCP_TOOL_FILTER="data"` | Verify only data + general + filesystem + object_builder tools registered | Config + mock |
| 11.7 | `all_prompt()` | Verify returns non-empty prompt string | **Pure logic** |
| 11.8 | `admin_prompt()` | Verify returns non-empty prompt string | **Pure logic** |
| 11.9 | `data_prompt()` | Verify returns non-empty prompt string | **Pure logic** |
| 11.10 | Transport config — STDIO (default) | Verify no HTTP settings when `MCP_TRANSPORT` unset | Config |
| 11.11 | Transport config — HTTP | Set `MCP_TRANSPORT=http` → verify host/port config | Config |

**Test file**: `tests/test_mcp_tools.py`

---

## 12. `scripts/setup_mcp.py` — Setup Script

| # | Function | Test Description | Category |
|---|---|---|---|
| 12.1 | `load_env_file()` — valid .env | Write test .env → verify key/value parsing | **Pure logic** (tmp file) |
| 12.2 | `load_env_file()` — comments and blanks | Verify comments/blank lines skipped | **Pure logic** |
| 12.3 | `load_env_file()` — quoted values | Verify quotes stripped from values | **Pure logic** |
| 12.4 | `load_env_file()` — export prefix | Verify `export KEY=val` handled | **Pure logic** |
| 12.5 | `load_env_file()` — missing file | Verify empty dict returned | **Pure logic** |
| 12.6 | `write_env_file()` — update existing | Pre-existing .env with KEY=old → write KEY=new → verify update | Tmp file |
| 12.7 | `write_env_file()` — add new key | Write new key to existing file → verify appended | Tmp file |
| 12.8 | `write_env_file()` — preserve comments | Verify comment lines untouched | Tmp file |
| 12.9 | `build_config_entry()` — VS Code STDIO | Verify `{"type":"stdio", "command":..., "args":[...]}` | **Pure logic** |
| 12.10 | `build_config_entry()` — VS Code HTTP | Verify `{"type":"http", "url":"http://host:port/mcp"}` | **Pure logic** |
| 12.11 | `build_config_entry()` — Cursor STDIO | Verify `{"command":..., "args":[...]}` (no type field) | **Pure logic** |
| 12.12 | `build_config_entry()` — Cursor HTTP | Verify `{"type":"http", "url":...}` | **Pure logic** |
| 12.13 | `get_vscode_config_dir()` — macOS | Mock `platform.system()="Darwin"` → verify `~/Library/Application Support/Code/User` | Mock platform |
| 12.14 | `get_vscode_config_dir()` — Linux | Mock `platform.system()="Linux"` → verify `~/.config/Code/User` | Mock platform |
| 12.15 | `get_vscode_config_dir()` — Windows | Mock `platform.system()="Windows"` + APPDATA → verify path | Mock platform |
| 12.16 | `get_cursor_config_dir()` — macOS | Verify `~/.cursor` | Mock platform |
| 12.17 | `is_virtual_environment_active()` — in venv | Mock `sys.prefix != sys.base_prefix` → verify `True` | Mock sys |
| 12.18 | `is_virtual_environment_active()` — not in venv | Verify `False` | Mock sys |
| 12.19 | `resolve_python_executable()` — valid command | Mock subprocess → verify resolved path | Mock subprocess |
| 12.20 | `resolve_python_executable()` — invalid command | Mock failed subprocess → verify `None` | Mock subprocess |
| 12.21 | `resolve_command_path()` — absolute path | Verify returned as-is | **Pure logic** |
| 12.22 | `resolve_command_path()` — relative path `./script` | Verify resolved against project_dir | **Pure logic** |
| 12.23 | `resolve_command_path()` — bare command | Verify returned as-is (not resolved) | **Pure logic** |
| 12.24 | `prompt_tool_filter()` | Mock `input()` → verify returns correct filter string | Mock input |
| 12.25 | `get_http_env_from_dotenv()` — all keys present | Verify returns dict with 3 keys | Pure logic + tmp file |
| 12.26 | `get_http_env_from_dotenv()` — missing keys | Verify returns `None` | Pure logic + tmp file |

**Test file**: `tests/test_setup_mcp.py`

---

## Summary Statistics

| Category | Test Count | Mocking Required |
|---|---|---|
| **Pure logic (no mocks)** | ~30 | None — data in, data out |
| **Filesystem/env mocks** | ~20 | `tmp_path`, `monkeypatch`, `os.environ` |
| **Evo SDK mock-required** | ~55 | `unittest.mock` / `pytest-mock` for SDK clients |
| **Async + mock** | ~50 | `pytest-asyncio` + mocked SDK |
| **Total** | **~155 test cases** | |

---

## Priority Order for Implementation

1. **`utils/evo_data_utils.py`** (5 tests) — Pure logic, zero dependencies, instant wins
2. **`utils/object_builders.py` — `build_bounding_box`, `build_hole_id_lookup`, `build_hole_index_map`** (~6 tests) — Pure logic methods in the largest file
3. **`scripts/setup_mcp.py` — `load_env_file`, `write_env_file`, `build_config_entry`, `resolve_command_path`** (~14 tests) — Pure logic, just string/file manipulation
4. **`evo_mcp/__init__.py`** (3 tests) — Version detection fallback chain
5. **`tools/filesystem_tools.py`** (11 tests) — Minimal SDK coupling, mostly filesystem ops
6. **`utils/object_builders.py` — remaining builder methods** (~22 tests) — Need `data_client` mock but highly testable
7. **`context.py`** (20 tests) — Core state management, requires SDK mocks
8. **`tools/general_tools.py`** (18 tests) — Standard mocked SDK pattern
9. **`tools/admin_tools.py`** (9 tests) — More complex workflows to mock
10. **`tools/object_build_tools.py`** (12 tests) — Integration of builders + SDK
11. **`tools/instance_users_admin_tools.py`** (10 tests) — Pagination logic worth testing
12. **`mcp_tools.py`** (11 tests) — Entry point configuration
13. **`tools/data_tools.py`** (4 tests) — Currently commented out, lowest priority

---

## Proposed Directory Structure

```
tests/
├── conftest.py                         # Shared fixtures (mock EvoContext, mock clients, tmp CSV files)
├── test_init.py                        # Package version detection
├── test_context.py                     # EvoContext class
├── test_mcp_tools.py                   # Entry point / server config
├── test_setup_mcp.py                   # Setup script
├── tools/
│   ├── conftest.py                     # Shared tool test fixtures (registered MCP, mock context)
│   ├── test_general_tools.py
│   ├── test_admin_tools.py
│   ├── test_data_tools.py
│   ├── test_filesystem_tools.py
│   ├── test_object_build_tools.py
│   └── test_instance_users_admin_tools.py
└── utils/
    ├── test_evo_data_utils.py
    └── test_object_builders.py
```
