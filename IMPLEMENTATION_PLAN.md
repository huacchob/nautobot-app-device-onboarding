# Implementation Plan: Port Power Supplies and Modules Functionality

## Overview
Port power supplies and modules functionality from branch `v443-power-supplies` (commit de489e1) to branch `modules-and-power-supplies`.

## Source Analysis
- **Source Branch:** v443-power-supplies
- **Target Branch:** modules-and-power-supplies
- **Key Commits:** 3214ac0 (backport), f2e0add (power supplies)
- **Nautobot Version:** 3.x (compatible with both branches)

## Implementation Goals

Each goal will result in one commit following the existing conventions.

---

### Goal 1: Add DiffSync Models for Modules
**Commit Message:** Add DiffSync models for module bay, module type, and module

**Files to Modify:**
- `nautobot_device_onboarding/diffsync/models/sync_network_data_models.py`

**Changes:**
1. Add imports for Module, ModuleBay, ModuleType, Manufacturer from nautobot.dcim.models
2. Add `SyncNetworkModuleBay` class with:
   - _modelname = "module_bay"
   - _identifiers: name, parent_device__name
   - create() method
   - delete() method (prevents deletion)

3. Add `SyncNetworkModuleType` class with:
   - _modelname = "module_type"
   - _identifiers: model, manufacturer__name
   - create() method
   - delete() method (prevents deletion)

4. Add `SyncNetworkModule` class with:
   - _modelname = "module"
   - _identifiers: module_type__model, module_type__manufacturer__name, parent_module_bay__name, parent_module_bay__parent_device__name
   - create() method
   - delete() method (prevents deletion)

**Type Annotations:**
- All parameters and return values must have type hints
- Use Optional[UUID] for pk fields
- Use proper typing for create() method

**Tests to Add:**
- None in this commit (will be added in later commit)

---

### Goal 2: Add Command Mappers for Modules
**Commit Message:** Add modules command mappers for Cisco IOS and XE

**Files to Modify:**
- `nautobot_device_onboarding/command_mappers/cisco_ios.yml`
- `nautobot_device_onboarding/command_mappers/cisco_xe.yml`

**Changes for Both Files:**
1. Add `modules` root key with commands:
   - Command: "show inventory"
   - Parser: textfsm
   - JPath filter to extract module names (excludes Power Supply, StackPort, interfaces)
   - Post-processor to convert to dict format

2. Add `modules__module_type`:
   - Command: "show inventory"
   - Extract PID for specific module
   - iterable_type: "str"

3. Add `modules__manufacturer`:
   - Command: "show inventory"
   - Post-processor returns "Cisco"
   - iterable_type: "str"

4. Add `modules__serial`:
   - Command: "show inventory"
   - Extract serial number
   - iterable_type: "str"

5. Add `modules__description`:
   - Command: "show inventory"
   - Extract description
   - iterable_type: "str"

**Tests to Add:**
- None in this commit (command mappers tested via integration tests)

---

### Goal 3: Add Modules Schema Definition
**Commit Message:** Add modules schema to network data schema

**Files to Modify:**
- `nautobot_device_onboarding/nornir_plays/schemas.py`

**Changes:**
1. Add `modules` key to NETWORK_DATA_SCHEMA with:
   - type: "object"
   - description: "Modules/line cards installed in the device"
   - items containing:
     - module_type (required, string)
     - manufacturer (required, string)
     - serial (optional, string)
     - description (optional, string)

**Tests to Add:**
- None in this commit (schema is validated implicitly)

---

### Goal 4: Add sync_modules Job Parameter
**Commit Message:** Add sync_modules parameter to SSOTSyncNetworkData job

**Files to Modify:**
- `nautobot_device_onboarding/jobs.py`

**Changes:**
1. Add `sync_modules` BooleanVar class variable:
   - default=False
   - description="Sync modules from device."

2. Add `sync_modules` to run() method signature (after sync_software_version)

3. Add `self.sync_modules = sync_modules` in run() method body

**Tests to Add:**
- None in this commit (tests updated in later commit)

---

### Goal 5: Update Command Getter for Modules
**Commit Message:** Update command getter to support modules sync flag

**Files to Modify:**
- `nautobot_device_onboarding/nornir_plays/command_getter.py`

**Changes:**
1. Update `_get_commands_to_run()` signature:
   - Add `sync_modules` parameter

2. Add filtering logic in both command processing blocks:
   - Skip commands where `key.startswith("modules")` if `not sync_modules`

3. Update `netmiko_send_commands()` call to `_get_commands_to_run()`:
   - Add `getattr(nautobot_job, "sync_modules", False)`

4. Update `sync_network_data_command_getter()`:
   - Add `sync_modules: job.sync_modules` to host data

**Type Annotations:**
- Update function signatures with proper type hints
- Ensure sync_modules: bool parameter

**Tests to Add:**
- Update all test_command_getter.py tests to include sync_modules=False

---

### Goal 6: Update Formatter to Skip Missing Commands
**Commit Message:** Update formatter to skip commands not in output

**Files to Modify:**
- `nautobot_device_onboarding/nornir_plays/formatter.py`

**Changes:**
1. In `perform_data_extraction()`, add check after loop_commands:
   - Skip if `show_command_dict["command"] not in command_outputs_dict`

**Tests to Add:**
- None in this commit (integration tested via job tests)

---

### Goal 7: Update Nautobot Adapter for Modules
**Commit Message:** Add module loading methods to Nautobot adapter

**Files to Modify:**
- `nautobot_device_onboarding/diffsync/adapters/sync_network_data_adapters.py`

**Changes:**
1. Add imports: Module, ModuleBay, ModuleType from nautobot.dcim.models

2. Add import: operator (for itemgetter)

3. Update SyncNetworkDataNautobotAdapter class:
   - Add to model definitions: module_bay, module_type, module
   - Add to top_level list: "module_bay", "module_type", "module"

4. Fix sorted() call in load_interfaces():
   - Change `sorted(tagged_vlans, key=lambda x: x["id"])`
   - To `sorted(tagged_vlans, key=operator.itemgetter("id"))`

5. Add `load_module_bay()` method:
   - Query all ModuleBay objects
   - Create DiffSync objects with SKIP_UNMATCHED_DST flag
   - Handle ObjectAlreadyExists exception

6. Add `load_module_type()` method:
   - Query all ModuleType objects
   - Create DiffSync objects with SKIP_UNMATCHED_DST flag
   - Handle ObjectAlreadyExists exception

7. Add `load_module()` method:
   - Query Module objects filtered by devices_to_load
   - Create DiffSync objects with SKIP_UNMATCHED_DST flag
   - Handle ObjectAlreadyExists exception

8. Update `load()` method:
   - Add conditional loading for module_bay if sync_modules
   - Add conditional loading for module_type if sync_modules
   - Add conditional loading for module if sync_modules

**Type Annotations:**
- Ensure all methods have proper type hints
- Parameters should specify types explicitly

**Tests to Add:**
- None in this commit (tests added in next commit)

---

### Goal 8: Update Network Adapter for Modules
**Commit Message:** Add module loading methods to Network adapter

**Files to Modify:**
- `nautobot_device_onboarding/diffsync/adapters/sync_network_data_adapters.py` (same file, continued)

**Changes:**
1. Update SyncNetworkDataNetworkAdapter class:
   - Add to model definitions: module_bay, module_type, module
   - Add to top_level list: "module_bay", "module_type", "module"

2. Fix sorted() call in load_tagged_vlans_to_interface():
   - Change `sorted(interface_data["tagged_vlans"], key=lambda x: x["id"])`
   - To `sorted(interface_data["tagged_vlans"], key=operator.itemgetter("id"))`

3. Add `load_module_bay()` method:
   - Iterate through command_getter_result
   - Extract module bay names from device_data["modules"].keys()
   - Create DiffSync objects with SKIP_UNMATCHED_DST flag
   - Handle exceptions with _handle_general_load_exception()

4. Add `load_module_type()` method:
   - Iterate through command_getter_result
   - Extract module_type and manufacturer from device_data["modules"].values()
   - Create DiffSync objects with SKIP_UNMATCHED_DST flag
   - Handle exceptions with _handle_general_load_exception()

5. Add `load_module()` method:
   - Iterate through command_getter_result
   - Extract full module data from device_data["modules"]
   - Create DiffSync objects with SKIP_UNMATCHED_DST flag
   - Handle exceptions with _handle_general_load_exception()

6. Update `load()` method:
   - Add conditional loading for module_bay if sync_modules
   - Add conditional loading for module_type if sync_modules
   - Add conditional loading for module if sync_modules

**Type Annotations:**
- Ensure all methods have proper type hints

**Tests to Add:**
- None in this commit (tests added in next commit)

---

### Goal 9: Add Test Fixtures for Modules
**Commit Message:** Add module test data to sync network data fixtures

**Files to Modify:**
- `nautobot_device_onboarding/tests/fixtures/sync_network_data_fixture.py`

**Changes:**
1. Add `modules` key to `sync_network_mock_data_valid["demo-cisco-1"]`:
   ```python
   "modules": {
       "Switch 1 FRU Uplink Module 1": {
           "module_type": "C9300-NM-4G",
           "manufacturer": "Cisco",
           "serial": "FJZ23271PFU",
           "description": "4x1G Uplink Module",
       },
   },
   ```

2. Add empty `modules` key to `sync_network_mock_data_valid["demo-cisco-2"]`:
   ```python
   "modules": {},
   ```

**Tests to Add:**
- None in this commit (fixtures support later tests)

---

### Goal 10: Update Command Getter Tests
**Commit Message:** Update command getter tests with sync_modules parameter

**Files to Modify:**
- `nautobot_device_onboarding/tests/test_command_getter.py`

**Changes:**
1. Update all `_get_commands_to_run()` calls to include:
   - `sync_modules=False` parameter

**Tests Affected:**
- test_get_commands_to_run_no_sync_vlans
- test_get_commands_to_run_with_no_sync_vlans
- test_get_commands_to_run_with_sync_vlans_and_sync_vrfs
- test_run_job_with_sync_vlans
- test_get_commands_to_run_with_sync_vrfs
- test_get_commands_to_run_with_sync_cables

**Tests to Add:**
- None new, just parameter updates

---

### Goal 11: Update Job Tests with Modules
**Commit Message:** Update job tests to include sync_modules parameter

**Files to Modify:**
- `nautobot_device_onboarding/tests/test_jobs.py`

**Changes:**
1. Update `test_data_source_sync_network_data` job data:
   - Add `"sync_modules": True`

2. Update `test_data_source_sync_network_data_check_device_location` job data:
   - Add `"sync_modules": True`

3. Update `test_data_source_sync_network_data_test_interface_create_check` job data:
   - Add `"sync_modules": False`

**Tests to Add:**
- Optionally add a specific test case for module syncing (if time permits)

---

### Goal 12: Update MEMORY.md and Create Handoff
**Commit Message:** Update MEMORY.md with implementation status

**Files to Modify:**
- `MEMORY.md`

**Changes:**
1. Update Current State section with completion status
2. Document what was implemented
3. Document any deviations from source
4. Add verification steps completed
5. Add notes for future maintainers

**Tests to Add:**
- None

---

## Testing Strategy

After implementation, run:

```bash
# Format code
poetry run invoke autoformat

# Run unit tests
poetry run invoke unittest

# Run full test suite
poetry run invoke tests
```

## Success Criteria

1. ✅ All code follows existing conventions
2. ✅ Type annotations present on all new code
3. ✅ All tests pass
4. ✅ Code passes ruff and pylint checks
5. ✅ Each goal = one commit with descriptive message
6. ✅ Documentation updated as needed
7. ✅ MEMORY.md updated with final status

## Risk Mitigation

- **Module models not available:** Verified Nautobot 3.x includes Module/ModuleBay/ModuleType
- **Test failures:** Each commit incrementally tested
- **Type annotation issues:** Follow existing patterns in codebase
- **Integration issues:** Fixtures match expected format from command mappers

## Notes

- Power supplies functionality mentioned in branch name but only modules implemented
- Command mappers filter out "Power Supply" in JPath, focusing on modules
- No HTML template changes needed (user indicated not to change)
- All changes are additive - no breaking changes to existing functionality
