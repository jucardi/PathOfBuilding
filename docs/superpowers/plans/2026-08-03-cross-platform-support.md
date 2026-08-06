# Cross-Platform Support (Phase 1: Repo Readiness) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make everything in this repository (Lua application, update pipeline, manifest tooling) platform-clean and multi-platform-capable, so that native Linux/macOS builds only require a ported native host and per-platform runtime bundles — no further application changes.

**Architecture:** Path of Building is a pure-Lua app (`src/`) running on a host API (specified by `src/HeadlessWrapper.lua`) provided by native binaries in `runtime/`. The update system (`src/UpdateCheck.lua` → ops files → `src/UpdateApply.lua`) is already platform-parameterised via `platform` attributes in `manifest.xml`, but the manifest generator hardcodes win32, the apply step has Windows-only assumptions (infinite file-lock retry, no executable-bit handling), and a few host-API call sites assume Windows-host-only functions. This plan fixes those, adds regression tests that run headlessly on any OS, and documents the platform model.

**Tech Stack:** Lua 5.1 (LuaJIT), busted (specs in `spec/`, config in `.busted`, run from repo root with cwd handling per `.busted` `directory = "src"`), Python 3.10+ + pytest (`tests/`), GitHub Actions.

## Global Constraints

- Lua code targets **LuaJIT / Lua 5.1**; no Lua 5.2+ constructs.
- Lua files use **tab indentation** (match surrounding code exactly).
- **Backward compatibility with deployed win32 clients is mandatory**: `manifest.xml` changes must keep existing clients updating correctly. Existing clients include a remote `<File>` iff it has no `platform` attribute or `platform` equals their local `<Version platform>` (`src/UpdateCheck.lua:158`), and select sources via `partSources[localPlatform] or partSources["any"]` (`src/UpdateCheck.lua:226`).
- **Do not touch line-ending behaviour**: `update_manifest.py` hashes CRLF-normalised bytes and `UpdateCheck.lua` compares `sha1(content)` OR `sha1(content:gsub("\n","\r\n"))`. Leave both as-is.
- **Never modify binaries in `runtime/`**; they are ingested from the SimpleGraphic repo by CI.
- Python code targets **Python 3.10.0+** (stated in `update_manifest.py:1`).
- Run Lua specs via the repo's Docker test harness from the repo root: `docker compose run --rm busted-tests busted --lua=luajit` (append `--filter="Name"` to scope). busted/luajit are not installed on this machine; the compose service mounts the repo read-only, so specs must only write outside the repo (use `os.tmpname()`-based temp dirs).
- Run Python tests from the repo root with the session venv: `/private/tmp/claude-502/-Users-jucardi-dev-thirdparty-PathOfBuilding/feec8ca8-a642-4b5e-82cb-f8ac7e58a430/scratchpad/venv/bin/python -m pytest tests/ -v` (system python3 has no pytest).
- Commit after every task; message prefixes follow repo convention (`fix:`, `feat:`, `test:`, `docs:`, `chore:`).

## File Structure

| File | Status | Responsibility |
|---|---|---|
| `update_manifest.py` | Modify | Manifest generator: read `part`/`platform` options from `manifest.cfg` sections; emit per-file `platform` attributes; include extensionless files |
| `manifest.cfg` | Modify | Declare `platform = win32` on `[runtime]` |
| `tests/test_update_manifest.py` | Create | pytest coverage for manifest generation |
| `src/UpdateApply.lua` | Modify | Bounded write-retry, `chmod` op, `{space}` handling in `chmod`, clear failure errors |
| `spec/System/TestUpdateApply_spec.lua` | Create | Op-interpreter specs (move/delete/chmod/start, failure path) |
| `src/UpdateCheck.lua` | Modify | Emit `chmod` ops for extensionless runtime files on non-win32 platforms |
| `spec/System/TestUpdateCheck_spec.lua` | Create | End-to-end update-check harness with fake curl/lzip; asserts generated ops per platform |
| `src/Launch.lua` | Modify | Guard `jit.opt.start` behind `if jit and jit.opt` |
| `src/Modules/Main.lua` | Modify | Guard `GetCloudProvider` call (optional host API) |
| `spec/System/TestCloudErrorPopup_spec.lua` | Create | Popup works without `GetCloudProvider` |
| `spec/System/TestAssetCase_spec.lua` | Create | CI guard: every `Assets/...` reference in Lua source matches an on-disk filename case-sensitively |
| `docs/crossPlatform.md` | Create | Platform model documentation |
| `CONTRIBUTING.md` | Modify | Fix stale `-PoE2.exe` filename; link platform doc |

---

### Task 1: Platform-aware manifest generation (`update_manifest.py`)

**Files:**
- Modify: `update_manifest.py:76-115`
- Modify: `manifest.cfg:6-9`
- Create: `tests/test_update_manifest.py`

**Interfaces:**
- Consumes: `manifest.cfg` (configparser INI). New optional per-section options: `part` (defaults to the section name) and `platform` (omitted = platform-neutral).
- Produces: `manifest.xml` where a section with `platform` set emits `<Source part="P" platform="X" url="..."/>` and every `<File>` from that section carries `platform="X"`. The legacy `runtime="win32"` File attribute (read by nothing) is dropped. Extensionless files (e.g. a POSIX `Update` binary) are now included by the glob. Later platform runtime dirs will be added as `[runtime-linux64]`-style sections with `part = runtime`.

- [ ] **Step 1: Write the failing tests**

Create `tests/test_update_manifest.py`:

```python
import pathlib
import sys
import xml.etree.ElementTree as Et

sys.path.insert(0, str(pathlib.Path(__file__).resolve().parent.parent))

from update_manifest import create_manifest


BASE_MANIFEST = (
    '<?xml version="1.0" encoding="UTF-8"?>\n'
    "<PoBVersion>\n"
    '\t<Version number="1.0.0" />\n'
    "</PoBVersion>\n"
)


def make_repo(tmp_path: pathlib.Path, cfg: str) -> None:
    (tmp_path / "manifest.xml").write_text(BASE_MANIFEST)
    (tmp_path / "manifest.cfg").write_text(cfg)
    runtime = tmp_path / "runtime"
    runtime.mkdir()
    (runtime / "SimpleGraphic.dll").write_bytes(b"\x00binary")
    (runtime / "Update").write_bytes(b"\x00posix-executable")  # extensionless
    lua = runtime / "lua"
    lua.mkdir()
    (lua / "xml.lua").write_text("-- lua module\n")
    src = tmp_path / "src"
    src.mkdir()
    (src / "Launch.lua").write_text("-- launch\n")


def generate(tmp_path, monkeypatch, cfg):
    make_repo(tmp_path, cfg)
    monkeypatch.chdir(tmp_path)
    create_manifest(version="1.2.3", replace=True)
    return Et.parse(tmp_path / "manifest.xml").getroot()


def test_platform_section_tags_sources_and_files(tmp_path, monkeypatch):
    root = generate(
        tmp_path,
        monkeypatch,
        "[runtime]\npath = runtime\nplatform = win32\n\n[program]\npath = src\n",
    )
    sources = {
        (s.get("part"), s.get("platform")): s.get("url") for s in root.findall("Source")
    }
    assert ("runtime", "win32") in sources
    assert ("program", None) in sources
    runtime_files = {
        f.get("name"): f for f in root.findall("File") if f.get("part") == "runtime"
    }
    # every file in a platformed section is tagged, not just .dll/.exe
    assert runtime_files["SimpleGraphic.dll"].get("platform") == "win32"
    assert runtime_files["lua/xml.lua"].get("platform") == "win32"
    # legacy attribute dropped
    assert runtime_files["SimpleGraphic.dll"].get("runtime") is None
    # extensionless files are included
    assert "Update" in runtime_files
    program_files = {
        f.get("name"): f for f in root.findall("File") if f.get("part") == "program"
    }
    assert program_files["Launch.lua"].get("platform") is None


def test_part_override_allows_multiple_runtime_sections(tmp_path, monkeypatch):
    cfg = (
        "[runtime]\npath = runtime\nplatform = win32\n\n"
        "[runtime-linux64]\npath = runtime\npart = runtime\nplatform = linux64\n"
    )
    root = generate(tmp_path, monkeypatch, cfg)
    sources = {(s.get("part"), s.get("platform")) for s in root.findall("Source")}
    assert ("runtime", "win32") in sources
    assert ("runtime", "linux64") in sources
    parts = {f.get("part") for f in root.findall("File")}
    assert parts == {"runtime"}
    platforms = {f.get("platform") for f in root.findall("File")}
    assert platforms == {"win32", "linux64"}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_update_manifest.py -v`
Expected: FAIL — `runtime_files["SimpleGraphic.dll"].get("platform") == "win32"` asserts `None == "win32"`, and `"Update"` is missing (current glob `**/*.*` skips extensionless files).

- [ ] **Step 3: Implement**

In `update_manifest.py`, replace the `parts` loop (lines 77-86):

```python
    parts: list[dict[str, str]] = []
    for section in config.sections():
        url = base_url + config[section]["path"]
        url_with_trailing_slash = url if url.endswith("/") else url + "/"
        part = config[section].get("part", section)
        platform = config[section].get("platform")
        attributes = {"part": part, "url": url_with_trailing_slash}
        if platform:
            attributes = {"part": part, "platform": platform, "url": url_with_trailing_slash}
        parts.append(attributes)
```

Replace the `files` loop (lines 88-115):

```python
    files: list[dict[str, str]] = []
    for section in config.sections():
        include_files = _parse_list_option(config, section, "include-files")
        include_dirs = _parse_list_option(config, section, "include-directories")
        exclude_files = _parse_list_option(config, section, "exclude-files")
        exclude_dirs = _parse_list_option(config, section, "exclude-directories")
        part = config[section].get("part", section)
        platform = config[section].get("platform")
        source = pathlib.Path(config[section]["path"])
        for path in source.glob("**/*"):
            if not path.is_file():
                continue
            if include_files and not _exclude_file(include_files, path):
                continue
            if include_dirs and not _exclude_directory(include_dirs, path):
                continue
            if exclude_files and _exclude_file(exclude_files, path):
                continue
            if exclude_dirs and _exclude_directory(exclude_dirs, path):
                continue
            data = path.read_bytes()
            # Normalize line endings for non-binary files in case they were accidentally mixed
            if b"\0" not in data:
                data = re.sub(rb"\r\n?|\n", b"\r\n", data)
            sha1 = hashlib.sha1(data).hexdigest()
            name = path.relative_to(config[section]["path"]).as_posix()
            attributes = {"name": name, "part": part, "sha1": sha1}
            if platform:
                attributes = {"name": name, "part": part, "platform": platform, "sha1": sha1}
            files.append(attributes)
```

In `manifest.cfg`, change the `[runtime]` section to declare its platform:

```ini
[runtime]
path = runtime
platform = win32
exclude-files = lua-profiler.lua,msvcr100.dll,SimpleGraphic.cfg,Update.exe,imgui.ini,SimpleGraphic.log
exclude-directories =
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/ -v`
Expected: PASS (both new tests plus the pre-existing `tests/test_fix_ascendancy_positions.py`).

- [ ] **Step 5: Verify against the real manifest**

Run: `python3 update_manifest.py --in-place && git diff --stat manifest.xml`
Then inspect: `git diff manifest.xml | head -80`
Expected changes ONLY of these kinds:
1. `<File ... runtime="win32" .../>` entries become `platform="win32"`.
2. Runtime files that previously had no per-file attribute (e.g. `lua/*.lua`, fonts) gain `platform="win32"`.
3. No file additions or removals (verify with `git diff manifest.xml | grep -c '^+.*<File'` ≈ `git diff manifest.xml | grep -c '^-.*<File'`).
If unexpected files appear (extensionless files previously missed by the glob outside `runtime/`), list them and add them to the appropriate `exclude-files` before committing.

- [ ] **Step 6: Commit**

```bash
git add update_manifest.py manifest.cfg manifest.xml tests/test_update_manifest.py
git commit -m "feat: platform-aware manifest generation for multi-platform runtimes"
```

---

### Task 2: POSIX-safe update apply (`src/UpdateApply.lua`)

**Files:**
- Modify: `src/UpdateApply.lua` (whole file, 47 lines)
- Create: `spec/System/TestUpdateApply_spec.lua`

**Interfaces:**
- Consumes: ops file with one op per line: `move "src" "dst"`, `delete "path"`, `start "path"` (existing) and `chmod "path"` (new).
- Produces: `chmod "path"` op contract — runs `chmod +x` on the (`{space}`-desanitised) path. Task 3's `UpdateCheck.lua` emits this op. Bounded dst-open retry (`maxOpenAttempts = 1000`) replaces the previous infinite loop; exhaustion raises a Lua error naming the destination.

- [ ] **Step 1: Write the failing spec**

Create `spec/System/TestUpdateApply_spec.lua`:

```lua
local lfs = require("lfs")

describe("UpdateApply", function()
	-- Use a temp dir outside the repo: the Docker test harness mounts the repo read-only
	local tmpDir = os.tmpname()
	os.remove(tmpDir)
	local originalSpawnProcess
	local originalExecute
	local spawned
	local executed

	local function writeFile(path, content)
		local file = assert(io.open(path, "wb"))
		file:write(content)
		file:close()
	end

	local function readFile(path)
		local file = io.open(path, "rb")
		if not file then
			return nil
		end
		local content = file:read("*a")
		file:close()
		return content
	end

	local function rmTree(path)
		if lfs.attributes(path, "mode") ~= "directory" then
			os.remove(path)
			return
		end
		for entry in lfs.dir(path) do
			if entry ~= "." and entry ~= ".." then
				rmTree(path.."/"..entry)
			end
		end
		lfs.rmdir(path)
	end

	local function runApply(ops)
		writeFile(tmpDir.."/opFile.txt", table.concat(ops, "\n"))
		return pcall(assert(loadfile("UpdateApply.lua")), tmpDir.."/opFile.txt")
	end

	before_each(function()
		rmTree(tmpDir)
		lfs.mkdir(tmpDir)
		spawned = { }
		executed = { }
		originalSpawnProcess = _G.SpawnProcess
		originalExecute = os.execute
		_G.SpawnProcess = function(target)
			table.insert(spawned, target)
		end
		os.execute = function(command)
			table.insert(executed, command)
			return 0
		end
	end)

	after_each(function()
		_G.SpawnProcess = originalSpawnProcess
		os.execute = originalExecute
		rmTree(tmpDir)
	end)

	it("moves files and desanitises {space} in the destination", function()
		writeFile(tmpDir.."/staged", "new content")
		local ok, err = runApply({ 'move "'..tmpDir..'/staged" "'..tmpDir..'/Path{space}of{space}Building"' })
		assert.is_true(ok, err)
		assert.are.equal("new content", readFile(tmpDir.."/Path of Building"))
		assert.is_nil(readFile(tmpDir.."/staged"))
	end)

	it("deletes files", function()
		writeFile(tmpDir.."/stale.lua", "old")
		local ok, err = runApply({ 'delete "'..tmpDir..'/stale.lua"' })
		assert.is_true(ok, err)
		assert.is_nil(readFile(tmpDir.."/stale.lua"))
	end)

	it("marks chmod targets executable, desanitising {space}", function()
		local ok, err = runApply({ 'chmod "'..tmpDir..'/Path{space}of{space}Building"' })
		assert.is_true(ok, err)
		assert.are.equal(1, #executed)
		assert.are.equal('chmod +x "'..tmpDir..'/Path of Building"', executed[1])
	end)

	it("starts processes", function()
		local ok, err = runApply({ 'start "'..tmpDir..'/Path of Building"' })
		assert.is_true(ok, err)
		assert.are.same({ tmpDir.."/Path of Building" }, spawned)
	end)

	it("raises a clear error instead of looping forever when the destination is unwritable", function()
		writeFile(tmpDir.."/staged", "new content")
		local ok, err = runApply({ 'move "'..tmpDir..'/staged" "'..tmpDir..'/no-such-dir/target"' })
		assert.is_false(ok)
		assert.matches("couldn't write", tostring(err))
	end)
end)
```

- [ ] **Step 2: Run spec to verify it fails**

Run: `busted --lua=luajit --filter="UpdateApply"`
Expected: FAIL — the `chmod` case reports `#executed` is 0 (unknown op is silently ignored today), and the unwritable-destination case hangs is prevented from being reached in old code only by the infinite loop; if the run appears to hang on that case, that confirms the bug — kill it and proceed. The other cases should pass.

- [ ] **Step 3: Implement**

Replace `src/UpdateApply.lua` (whole file) with:

```lua
#@
-- Path of Building
--
-- Module: Update Apply
-- Applies updates.
--
local opFileName = ...

local maxOpenAttempts = 1000

print("Applying update...")
local opFile = io.open(opFileName, "r")
if not opFile then
	print("No operations list present.\n")
	return
end
local lines = { }
for line in opFile:lines() do
	table.insert(lines, line)
end
opFile:close()
os.remove(opFileName)
for _, line in ipairs(lines) do
	local op, args = line:match("(%a+) ?(.*)")
	if op == "move" then
		local src, dst = args:match('"(.*)" "(.*)"')
		dst = dst:gsub("{space}", " ")
		print("Updating '"..dst.."'")
		local srcFile = io.open(src, "rb")
		assert(srcFile, "couldn't open "..src)
		local dstFile, openErr
		-- The destination may be transiently locked (e.g. antivirus on Windows); retry, but bounded
		for _ = 1, maxOpenAttempts do
			dstFile, openErr = io.open(dst, "w+b")
			if dstFile then
				break
			end
		end
		assert(dstFile, "couldn't write "..dst..(openErr and (": "..openErr) or ""))
		dstFile:write(srcFile:read("*a"))
		dstFile:close()
		srcFile:close()
		os.remove(src)
	elseif op == "delete" then
		local file = args:match('"(.*)"')
		print("Deleting '"..file.."'")
		os.remove(file)
	elseif op == "chmod" then
		local file = args:match('"(.*)"'):gsub("{space}", " ")
		print("Marking '"..file.."' as executable")
		os.execute('chmod +x "'..file..'"')
	elseif op == "start" then
		local target = args:match('"(.*)"')
		SpawnProcess(target)
	end
end
```

- [ ] **Step 4: Run spec to verify it passes**

Run: `busted --lua=luajit --filter="UpdateApply"`
Expected: 5 successes, 0 failures.

- [ ] **Step 5: Run the full suite to check for regressions**

Run: `busted --lua=luajit`
Expected: PASS (same pass count as `dev` baseline plus 5).

- [ ] **Step 6: Commit**

```bash
git add src/UpdateApply.lua spec/System/TestUpdateApply_spec.lua
git commit -m "feat: bounded retries, chmod op, and clear errors in update apply"
```

---

### Task 3: Emit executable-bit ops from update check (`src/UpdateCheck.lua`)

**Files:**
- Modify: `src/UpdateCheck.lua:313-321`
- Create: `spec/System/TestUpdateCheck_spec.lua`

**Interfaces:**
- Consumes: the `chmod "path"` op implemented in Task 2 (`{space}`-sanitised paths allowed).
- Produces: for platforms other than `win32`, every updated `part="runtime"` file whose basename has no extension (matches POSIX executables like `Path{space}of{space}Building` and `Update`) gets a `chmod` op appended right after its `move` op in `Update/opFileRuntime.txt`. Win32 behaviour is byte-identical to today.

- [ ] **Step 1: Write the failing spec**

Create `spec/System/TestUpdateCheck_spec.lua`. This is an end-to-end harness: it fabricates local + remote manifests, fakes `lcurl.safe`/`lzip` via a `require` shim, points the path helpers at a temp dir, runs the real `UpdateCheck.lua`, and asserts on the generated ops files.

```lua
local lfs = require("lfs")
local sha1 = require("sha1")

describe("UpdateCheck", function()
	-- Use a temp dir outside the repo: the Docker test harness mounts the repo read-only
	local tmpDir = os.tmpname()
	os.remove(tmpDir)
	local originalRequire
	local originalMakeDir
	local originalGetScriptPath
	local originalGetRuntimePath

	local programContent = "-- new launch script\n"
	local runtimeContent = "\0new runtime binary"
	local soContent = "\0new shared object"

	local function writeFile(path, content)
		local file = assert(io.open(path, "wb"))
		file:write(content)
		file:close()
	end

	local function readFile(path)
		local file = io.open(path, "rb")
		if not file then
			return nil
		end
		local content = file:read("*a")
		file:close()
		return content
	end

	local function rmTree(path)
		if lfs.attributes(path, "mode") ~= "directory" then
			os.remove(path)
			return
		end
		for entry in lfs.dir(path) do
			if entry ~= "." and entry ~= ".." then
				rmTree(path.."/"..entry)
			end
		end
		lfs.rmdir(path)
	end

	local function newFakeCurl(responses)
		local curl = { OPT_ACCEPT_ENCODING = 0, OPT_IPRESOLVE = 1, OPT_PROXY = 2, OPT_SSL_VERIFYPEER = 3, OPT_SSL_VERIFYHOST = 4 }
		function curl.easy()
			local easy = { url = "" }
			function easy:escape(text)
				return text
			end
			function easy:setopt_url(url)
				self.url = url
			end
			function easy:setopt()
			end
			function easy:setopt_writefunction(sink)
				self.sink = sink
			end
			function easy:perform()
				local content = responses[self.url]
				if not content then
					return nil, { msg = function() return "404: "..self.url end }
				end
				if type(self.sink) == "function" then
					self.sink(content)
				else
					self.sink:write(content)
				end
				return true, nil
			end
			function easy:close()
			end
			return easy
		end
		return curl
	end

	-- Builds a local manifest on disk and returns the canned remote responses
	local function setUpManifests(platform)
		writeFile(tmpDir.."/manifest.xml", table.concat({
			'<?xml version="1.0" encoding="UTF-8"?>',
			'<PoBVersion>',
			'\t<Version number="1.0.0" platform="'..platform..'" branch="dev" />',
			'\t<Source part="default" url="http://fake/" />',
			'\t<File name="Launch.lua" part="program" sha1="0000000000000000000000000000000000000000" />',
			'\t<File name="Path{space}of{space}Building" part="runtime" platform="'..platform..'" sha1="1111111111111111111111111111111111111111" />',
			'\t<File name="SimpleGraphic.so" part="runtime" platform="'..platform..'" sha1="2222222222222222222222222222222222222222" />',
			'</PoBVersion>',
		}, "\n"))
		local remoteManifest = table.concat({
			'<?xml version="1.0" encoding="UTF-8"?>',
			'<PoBVersion>',
			'\t<Version number="1.0.1" />',
			'\t<Source part="default" url="http://fake/" />',
			'\t<Source part="program" url="http://fake/prog/" />',
			'\t<Source part="runtime" platform="'..platform..'" url="http://fake/rt/" />',
			'\t<File name="Launch.lua" part="program" sha1="'..sha1(programContent)..'" />',
			'\t<File name="Path{space}of{space}Building" part="runtime" platform="'..platform..'" sha1="'..sha1(runtimeContent)..'" />',
			'\t<File name="SimpleGraphic.so" part="runtime" platform="'..platform..'" sha1="'..sha1(soContent)..'" />',
			'</PoBVersion>',
		}, "\n")
		return {
			["http://fake/manifest.xml"] = remoteManifest,
			["http://fake/changelog.txt"] = "changelog",
			["http://fake/prog/Launch.lua"] = programContent,
			["http://fake/rt/Path{space}of{space}Building"] = runtimeContent,
			["http://fake/rt/SimpleGraphic.so"] = soContent,
		}
	end

	local function runUpdateCheck(platform)
		local responses = setUpManifests(platform)
		local fakeCurl = newFakeCurl(responses)
		_G.require = function(name)
			if name == "lcurl.safe" then
				return fakeCurl
			elseif name == "lzip" then
				return { }
			end
			return originalRequire(name)
		end
		return assert(loadfile("UpdateCheck.lua"))()
	end

	before_each(function()
		rmTree(tmpDir)
		lfs.mkdir(tmpDir)
		lfs.mkdir(tmpDir.."/runtime")
		originalRequire = _G.require
		originalMakeDir = _G.MakeDir
		originalGetScriptPath = _G.GetScriptPath
		originalGetRuntimePath = _G.GetRuntimePath
		_G.GetScriptPath = function()
			return tmpDir
		end
		_G.GetRuntimePath = function()
			return tmpDir.."/runtime"
		end
		_G.MakeDir = function(path)
			if path:sub(1, 1) ~= "/" then
				path = tmpDir.."/"..path
			end
			lfs.mkdir(path)
			return true
		end
	end)

	after_each(function()
		_G.require = originalRequire
		_G.MakeDir = originalMakeDir
		_G.GetScriptPath = originalGetScriptPath
		_G.GetRuntimePath = originalGetRuntimePath
		rmTree(tmpDir)
	end)

	it("stages runtime updates with chmod ops for extensionless files on linux64", function()
		local mode = runUpdateCheck("linux64")
		assert.are.equal("basic", mode)
		local opsRuntime = assert(readFile(tmpDir.."/Update/opFileRuntime.txt"))
		assert.is_truthy(opsRuntime:find('move "'..tmpDir..'/Update/Path{space}of{space}Building" "'..tmpDir..'/runtime/Path{space}of{space}Building"', 1, true))
		assert.is_truthy(opsRuntime:find('chmod "'..tmpDir..'/runtime/Path{space}of{space}Building"', 1, true))
		-- files with extensions never get chmod
		assert.is_falsy(opsRuntime:find('chmod "'..tmpDir..'/runtime/SimpleGraphic.so"', 1, true))
		assert.is_truthy(opsRuntime:find('start "'..tmpDir..'/runtime/Path of Building"', 1, true))
		local ops = assert(readFile(tmpDir.."/Update/opFile.txt"))
		assert.is_truthy(ops:find('move "'..tmpDir..'/Update/Launch.lua" "'..tmpDir..'/Launch.lua"', 1, true))
	end)

	it("emits no chmod ops on win32", function()
		local mode = runUpdateCheck("win32")
		assert.are.equal("basic", mode)
		local opsRuntime = assert(readFile(tmpDir.."/Update/opFileRuntime.txt"))
		assert.is_falsy(opsRuntime:find("chmod", 1, true))
	end)
end)
```

- [ ] **Step 2: Run spec to verify it fails**

Run: `busted --lua=luajit --filter="UpdateCheck"`
Expected: the linux64 case FAILS on the `chmod` assertion (no chmod op emitted yet); the win32 case PASSES. If either case fails earlier (e.g. in the fake-curl plumbing), fix the spec first until the only failure is the missing `chmod` op.

- [ ] **Step 3: Implement**

In `src/UpdateCheck.lua`, replace lines 313-321:

```lua
		if data.part == "runtime" then
			-- Core runtime file, will need to update from the basic environment
			-- These files will be updated on the second pass of the update script, with the first pass being run within the normal environment
			updateMode = "basic"
			table.insert(opsRuntime, 'move "'..data.updateFileName..'" "'..data.fullPath..'"')
			if localPlatform ~= "win32" and not data.name:match("%.[^/]+$") then
				-- POSIX executables ship without an extension and lose their executable bit when rewritten
				table.insert(opsRuntime, 'chmod "'..data.fullPath..'"')
			end
		else
			table.insert(ops, 'move "'..data.updateFileName..'" "'..data.fullPath..'"')
		end
```

- [ ] **Step 4: Run spec to verify it passes**

Run: `busted --lua=luajit --filter="UpdateCheck"`
Expected: 2 successes, 0 failures.

- [ ] **Step 5: Run the full suite**

Run: `busted --lua=luajit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/UpdateCheck.lua spec/System/TestUpdateCheck_spec.lua
git commit -m "feat: preserve executable bits when updating POSIX runtimes"
```

---

### Task 4: Guard optional host APIs (`jit.opt`, `GetCloudProvider`)

**Files:**
- Modify: `src/Launch.lua:17`
- Modify: `src/Modules/Main.lua:1723-1725`
- Create: `spec/System/TestCloudErrorPopup_spec.lua`

**Interfaces:**
- Consumes: `main:OpenCloudErrorPopup(fileName)` (existing), global `GetCloudProvider` (host-optional after this task).
- Produces: the app runs on hosts that do not provide `GetCloudProvider` (e.g. pobfrontend, future POSIX hosts) or a `jit` global. No signature changes.

- [ ] **Step 1: Write the failing spec**

Create `spec/System/TestCloudErrorPopup_spec.lua`:

```lua
describe("OpenCloudErrorPopup", function()
	it("works when the host does not provide GetCloudProvider", function()
		local originalGetCloudProvider = _G.GetCloudProvider
		_G.GetCloudProvider = nil
		local ok, err = pcall(function()
			main:OpenCloudErrorPopup("SomeBuild.xml")
		end)
		_G.GetCloudProvider = originalGetCloudProvider
		if ok then
			main:ClosePopup()
		end
		assert.is_true(ok, tostring(err))
	end)
end)
```

- [ ] **Step 2: Run spec to verify it fails**

Run: `busted --lua=luajit --filter="OpenCloudErrorPopup"`
Expected: FAIL with "attempt to call global 'GetCloudProvider' (a nil value)".

- [ ] **Step 3: Implement**

In `src/Modules/Main.lua`, replace line 1724 (`local provider, _, status = GetCloudProvider(fileName)`) with:

```lua
	local provider, _, status
	if GetCloudProvider then
		provider, _, status = GetCloudProvider(fileName)
	end
```

In `src/Launch.lua`, replace line 17 (`jit.opt.start('maxtrace=4000','maxmcode=8192')`) with:

```lua
if jit and jit.opt then
	jit.opt.start('maxtrace=4000','maxmcode=8192')
end
```

- [ ] **Step 4: Run spec to verify it passes**

Run: `busted --lua=luajit --filter="OpenCloudErrorPopup"`
Expected: PASS.

- [ ] **Step 5: Run the full suite (exercises Launch.lua via HeadlessWrapper)**

Run: `busted --lua=luajit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/Launch.lua src/Modules/Main.lua spec/System/TestCloudErrorPopup_spec.lua
git commit -m "fix: tolerate hosts without GetCloudProvider or LuaJIT opt API"
```

---

### Task 5: Asset filename case-sensitivity guard

Windows is case-insensitive, so `Assets/` references with wrong casing work there but break on Linux/macOS (this class of bug has recurred — see commit `2b76d3406`). Add a spec that fails on any mismatch, then fix current offenders.

**Files:**
- Create: `spec/System/TestAssetCase_spec.lua`
- Modify: any `src/**.lua` files the spec reports (fix the *reference* to match the on-disk name — renaming shipped asset files would churn `manifest.xml` and break in-flight updates)

**Interfaces:**
- Consumes: on-disk names under `src/Assets/` (cwd is `src` when specs run), string literals matching `Assets/<path>` in Lua source.
- Produces: a permanent CI guard; no code interface.

- [ ] **Step 1: Write the spec**

Create `spec/System/TestAssetCase_spec.lua`:

```lua
local lfs = require("lfs")

describe("Asset references", function()
	-- Collect actual files under a directory with their exact on-disk casing
	local function collectFiles(dir, prefix, out)
		out = out or { }
		for entry in lfs.dir(dir) do
			if entry ~= "." and entry ~= ".." then
				local full = dir.."/"..entry
				local rel = prefix..entry
				if lfs.attributes(full, "mode") == "directory" then
					collectFiles(full, rel.."/", out)
				else
					out[rel] = true
				end
			end
		end
		return out
	end

	-- Collect Lua sources, skipping data/export dirs that don't reference assets
	local function collectLuaFiles(dir, out)
		out = out or { }
		for entry in lfs.dir(dir) do
			if entry ~= "." and entry ~= ".." then
				local full = dir.."/"..entry
				local mode = lfs.attributes(full, "mode")
				if mode == "directory" then
					if entry ~= "Data" and entry ~= "TreeData" and entry ~= "Export" and entry ~= "Builds" then
						collectLuaFiles(full, out)
					end
				elseif entry:match("%.lua$") then
					table.insert(out, full)
				end
			end
		end
		return out
	end

	it("match on-disk filenames exactly (case-sensitive)", function()
		local actual = collectFiles("Assets", "Assets/")
		local lowerToActual = { }
		for name in pairs(actual) do
			lowerToActual[name:lower()] = name
		end
		local mismatches = { }
		for _, luaFile in ipairs(collectLuaFiles(".")) do
			local file = assert(io.open(luaFile, "rb"))
			local content = file:read("*a")
			file:close()
			for ref in content:gmatch('"(Assets/[%w_%-%./]+)"') do
				if not actual[ref] then
					local hint = lowerToActual[ref:lower()]
					table.insert(mismatches, string.format("%s references %q%s",
						luaFile, ref, hint and (" (on disk: %q)"):format(hint) or " (no such file)"))
				end
			end
		end
		assert.are.equal(0, #mismatches, "\n"..table.concat(mismatches, "\n"))
	end)
end)
```

- [ ] **Step 2: Run the spec**

Run: `busted --lua=luajit --filter="Asset references"`
Expected: either PASS (no current mismatches — proceed to Step 4) or FAIL with a list of `file references "Assets/Foo.png" (on disk: "Assets/foo.png")` lines.

- [ ] **Step 3: Fix every reported mismatch**

For each reported line, edit the Lua source and change the string literal to the exact on-disk casing shown in the hint. If a reference reports `(no such file)`, investigate before changing anything — it may be a dynamically-constructed path the regex caught partially; if so, tighten the spec's pattern rather than deleting the reference. Re-run the spec after each batch of edits.

- [ ] **Step 4: Run spec + full suite to verify**

Run: `busted --lua=luajit`
Expected: PASS, including "Asset references".

- [ ] **Step 5: Commit**

```bash
git add spec/System/TestAssetCase_spec.lua src/
git commit -m "test: enforce case-sensitive asset references for POSIX filesystems"
```

---

### Task 6: Document the platform model

**Files:**
- Create: `docs/crossPlatform.md`
- Modify: `CONTRIBUTING.md:70` (stale filename) and the Linux section around `CONTRIBUTING.md:206`

**Interfaces:**
- Consumes: everything established in Tasks 1-5.
- Produces: documentation only.

- [ ] **Step 1: Create `docs/crossPlatform.md`**

```markdown
# Cross-platform architecture

Path of Building is a pure-Lua application (`src/`) that runs on a native host.
The host API contract is specified, in executable form, by
[`src/HeadlessWrapper.lua`](../src/HeadlessWrapper.lua): any host that provides
those globals (rendering, input, filesystem search, clipboard, subscripts,
`Inflate`/`Deflate`, path helpers) can run the app. The shipping host is
SimpleGraphic (built from
[PathOfBuilding-SimpleGraphic](https://github.com/PathOfBuildingCommunity/PathOfBuilding-SimpleGraphic)),
which renders through GLFW + ANGLE (OpenGL ES) and is delivered into `runtime/`
by the `update-simple-graphic` workflow.

## Platform identity

A client learns its platform from the `platform` attribute of the `<Version>`
element in its local `manifest.xml` (e.g. `win32`). The updater
(`src/UpdateCheck.lua`) then:

- includes a remote `<File>` iff it has no `platform` attribute or its
  `platform` matches the local platform;
- downloads each part from `<Source part="..." platform="...">` matching the
  local platform, falling back to the platform-less source.

`update_manifest.py` generates these attributes from `manifest.cfg`: a section
with a `platform` option tags its source and every file it contains with that
platform. A section may set `part` to publish under a shared part name, so a
future `[runtime-linux64]` section (with `part = runtime`,
`platform = linux64`) ships an alternative runtime bundle without any client
code changes.

## Update ops

`UpdateCheck.lua` stages downloads and writes an ops file that
`UpdateApply.lua` executes (`move`, `delete`, `chmod`, `start`). On non-win32
platforms, updated runtime files without a file extension (the POSIX
executables) get a `chmod` op so they stay executable after being rewritten.
Runtime files are applied by a second, minimal host (`runtime/Update` /
`Update.exe`) because the main host's own binaries cannot replace themselves
while running.

## Host expectations

Hosts are not required to provide every global: `GetCloudProvider` is optional,
and `jit.opt` tuning is skipped when unavailable. Asset paths are
case-sensitive on Linux/macOS; `spec/System/TestAssetCase_spec.lua` enforces
that all `Assets/` references match on-disk casing exactly.

## Status

Native Linux/macOS support additionally requires (tracked as follow-on plans):

1. A POSIX/macOS system layer in PathOfBuilding-SimpleGraphic publishing
   `SimpleGraphicDLLs-<arch>-<os>.tar` release assets (the Windows asset
   already follows this naming).
2. Per-platform runtime bundles in this repo (`[runtime-<platform>]` manifest
   sections), ingestion workflow updates, and packaging (tar.gz, then
   AppImage/dmg). Until then, Linux users run the Windows build under Wine or
   use community hosts such as pobfrontend.
```

- [ ] **Step 2: Fix CONTRIBUTING.md**

At `CONTRIBUTING.md:70`, change `./runtime/Path{space}of{space}Building-PoE2.exe` to `./runtime/Path{space}of{space}Building.exe` (the `-PoE2` name does not exist in this repo).

In the Linux section (around `CONTRIBUTING.md:206`), after the Wine instructions, add:

```markdown
See [docs/crossPlatform.md](docs/crossPlatform.md) for how platform support is
structured and what native Linux/macOS support requires.
```

- [ ] **Step 3: Verify docs render and links resolve**

Run: `grep -n "crossPlatform" CONTRIBUTING.md docs/crossPlatform.md && ls docs/crossPlatform.md src/HeadlessWrapper.lua`
Expected: both references print; both files exist.

- [ ] **Step 4: Commit**

```bash
git add docs/crossPlatform.md CONTRIBUTING.md
git commit -m "docs: document the cross-platform architecture and platform model"
```

---

## Execution decisions (recorded 2026-08-05)

- Executed on branch `feat/crossplatform` (8 commits, full suite 472/472 green, final whole-branch review: ready to merge).
- **Accepted tradeoff (user decision):** the bounded update-apply retry (`maxOpenAttempts = 1000`) completes in milliseconds, so a Windows file lock that outlasts it now fails the update visibly (staged files are reused on the next attempt) instead of hanging indefinitely as before. Fast-fail accepted over silent hang.
- Deferred to Plan B/C: shell-quoting hardening of the `chmod` op (`os.execute` single-quote escaping) — becomes reachable only when a POSIX runtime actually ships.

## Follow-on plans (separate documents, not part of this plan)

**Plan B — SimpleGraphic POSIX/macOS port** (in the `PathOfBuilding-SimpleGraphic` repo; requires cloning it):
Implement `engine/system/posix/` (and macOS specifics) alongside the existing `engine/system/win/`: window glue (GLFW already abstracts most), clipboard, `SpawnProcess`, `NewFileSearch`, user-data paths (XDG / `~/Library/Application Support`), subscript threading, plus a POSIX build of the minimal `Update` host. Publish `SimpleGraphicDLLs-x64-linux.tar` / `SimpleGraphicDLLs-arm64-macos.tar` release assets mirroring the existing `-x64-windows` naming. The API contract to satisfy is `src/HeadlessWrapper.lua` in this repo. On macOS, ANGLE-on-Metal covers rendering; notarization/Gatekeeper constraints on the self-updater need a design decision (likely: disable runtime self-update on macOS initially, app-bundle updates via download).

**Plan C — Runtime ingestion and packaging** (this repo; blocked on Plan B artifacts):
Add `runtime-linux64/` (and macOS) directories populated by extending `.github/workflows/update-simple-graphic.yml` to download the new per-platform tars; add `[runtime-linux64]`-style sections to `manifest.cfg` using Task 1's `part`/`platform` options; regenerate `manifest.xml`; add release packaging jobs producing portable `tar.gz` bundles first, then AppImage/dmg; register the `pob:` protocol handler per platform; update README/CONTRIBUTING install docs.
