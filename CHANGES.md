# Remaining changes (loadpackwidget.h/.cpp are provided as full files)

## packmanager.h

Add `#include <vector>`. In the public section:

```cpp
struct RemoteVersion {
    std::string version;
    std::string url;
    std::string sha256;
    std::vector<std::string> changelog;
};
typedef std::function<void(bool, const std::vector<RemoteVersion>&)> versions_callback;

bool hasRepositories() const { return !_repositories.empty(); }
void clearRepositories() { _repositories.clear(); _packs = json::object(); }
/// Fetches and validates a pack list; cb(true) if it is a valid pack list.
void testRepository(const std::string& url, std::function<void(bool)> cb);
/// Lists all downloadable versions of a pack from its versions.json (newest first as in the file).
void getPackVersions(const std::string& uid, versions_callback cb);
```

In the private section:

```cpp
void resolveVersionsUrl(const std::string& uid, const std::string& versions_url,
                        std::function<void(const std::string&)> cb); // cb("") on failure/denied
```

## packmanager.cpp

Replace steps 2-5 of `checkForUpdate` with a shared helper (this also fixes a hang: if the
user had already answered "no" for a host, neither `ncb` nor anything else was called):

```cpp
bool PackManager::checkForUpdate(const std::string& uid, const std::string& version, const std::string& versions_url,
        update_available_callback cb, no_update_available_callback ncb)
{
    if (_tempIgnoredSourceVersion[uid].count(version)) {
        printf("PackManager: Skipping update check for %s %s\n",
                sanitize_print(uid).c_str(), sanitize_print(version).c_str());
        if (ncb) ncb(uid);
        return false;
    }
    printf("PackManager: Checking for update for %s\n", sanitize_print(uid).c_str());
    resolveVersionsUrl(uid, versions_url, [this, uid, version, cb, ncb](const std::string& url) {
        if (url.empty()) {
            if (ncb) ncb(uid);
            return;
        }
        checkForUpdateFrom(uid, version, url, cb, ncb);
    });
    return true; // true = process started
}

void PackManager::resolveVersionsUrl(const std::string& uid, const std::string& versions_url,
        std::function<void(const std::string&)> cb)
{
    // 1. update repositories
    updateRepositories([this, uid, versions_url, cb](bool) {
        // 2. determine which versions_url to use
        std::string url = versions_url;
        auto it = _packs.find(uid);
        if (it != _packs.end() && it.value()["versions_url"].is_string()) {
            url = it.value()["versions_url"].get<std::string>();
            if (!HTTP::is_uri(url)) {
                fprintf(stderr, "WARNING: invalid versions_url in pack!");
                url = versions_url;
            }
        }
        if (url.empty()) {
            printf("Nowhere to check for updates of %s\n", sanitize_print(uid).c_str());
            cb("");
            return;
        }
        // 3. if versions url is not secure, bail out
        std::string proto, host, port, path;
        if (!HTTP::parse_uri(url, proto, host, port, path)) {
            printf("Invalid versions URL: %s\n", sanitize_print(url).c_str());
            cb("");
            return;
        }
        if (proto != "https" && (proto != "http" || host != "localhost" || (port != "" && port != "80"))) {
            printf("Unsupported URL: %s. Please use HTTPS.\n", sanitize_print(url).c_str());
            cb("");
            return;
        }
        // 4. if host is not trusted, ask user
        if (_trustedHosts.find(host) != _trustedHosts.end()) {
            cb(url);
        } else if (_untrustedHosts.find(host) == _untrustedHosts.end()) {
            askConfirmation("Load update information from " + host + "?", [this, url, host, cb](bool res) {
                if (res) {
                    _trustedHosts.insert(host);
                    _conf["trusted_hosts"] = _trustedHosts;
                    saveConf();
                    cb(url);
                } else {
                    _untrustedHosts.insert(host); // don't ask again until restart
                    cb("");
                }
            });
        } else {
            cb("");
        }
    });
}

void PackManager::getPackVersions(const std::string& uid, versions_callback cb)
{
    resolveVersionsUrl(uid, "", [this, cb](const std::string& url) {
        if (url.empty()) {
            cb(false, std::vector<RemoteVersion>{});
            return;
        }
        GetCached(url, [this, url, cb](bool ok, std::string data) {
            std::vector<RemoteVersion> out;
            if (ok) {
                try {
                    auto j = json::parse(data);
                    valijson::Validator validator;
                    JsonSchemaAdapter jAdapter(j);
                    if (!validator.validate(_versionSchema, jAdapter, nullptr))
                        throw std::runtime_error("Json validation failed");
                    for (auto& section : j["versions"]) {
                        if (!section["download_url"].is_string())
                            continue; // "null" = no download for this version
                        RemoteVersion v;
                        v.version = section["package_version"].get<std::string>();
                        v.url = section["download_url"].get<std::string>();
                        v.sha256 = section["sha256"].get<std::string>(); // required by schema if url is set
                        if (v.version.empty() || v.url.empty() || v.sha256.empty())
                            continue;
                        for (const auto& line : section["changelog"])
                            if (line.is_string()) v.changelog.push_back(line.get<std::string>());
                        out.push_back(std::move(v));
                    }
                } catch (...) {
                    printf("Invalid json versions %s\n", sanitize_print(url).c_str());
                    ok = false;
                    out.clear();
                }
            }
            cb(ok, out);
        });
    });
}

void PackManager::testRepository(const std::string& url, std::function<void(bool)> cb)
{
    GetCached(url, [this, cb](bool ok, std::string data) {
        bool valid = false;
        if (ok) {
            try {
                auto j = json::parse(data);
                valijson::Validator validator;
                JsonSchemaAdapter jAdapter(j);
                valid = validator.validate(_packsSchema, jAdapter, nullptr);
            } catch (...) {}
        }
        cb(valid);
    });
}
```

## defaulttrackerwindow.h

```cpp
Signal<const std::string&, const std::string&, const std::string&, const std::string&> onRemotePackVersionSelected;
```

## defaulttrackerwindow.cpp

After the existing `_loadPackWidget->onPackSelected += ...` block:

```cpp
_loadPackWidget->onRemoteVersionSelected += {this, [this](void*, const std::string& uid,
        const std::string& version, const std::string& url, const std::string& sha256) {
    onRemotePackVersionSelected.emit(this, uid, version, url, sha256);
}};
```

and change `refreshPackList()` to call `_loadPackWidget->updateInstalled();`.

## poptracker.h

Add to the private section: `void promptRemotePackList();`

## poptracker.cpp

### 1. Constructor: no more hard-coded repository

Replace the `_packManager->addRepository("https://raw.githubusercontent...")` line and the
TODO above it with (NOTE: use `find`, never `_config["remote_pack_list_url"]` - `operator[]`
would insert a null entry and break the "was the user asked yet?" check):

```cpp
{
    // "remote_pack_list_url": absent = never asked, "" = user declined, otherwise the URL
    const auto it = _config.find("remote_pack_list_url");
    if (it != _config.end() && it->is_string() && !it->get<std::string>().empty()) {
        if (!_packManager->addRepository(it->get<std::string>()))
            fprintf(stderr, "Warning: invalid remote_pack_list_url in config (https:// required)\n");
    }
}
```

### 2. First-start prompt

```cpp
void PopTracker::promptRemotePackList()
{
    static constexpr char COMMUNITY_URL[] =
        "https://raw.githubusercontent.com/poptracker-community/community-packlist/main/community-packs.json";

    if (!Dlg::hasGUI()) return; // can't ask; try again next start

    auto declined = [this]() {
        _config["remote_pack_list_url"] = ""; // remember that we asked
        saveConfig();
        Dlg::MsgBox("PopTracker",
                "No remote pack list was added.\n\n"
                "To add one later, set \"remote_pack_list_url\" in PopTracker.json "
                "to the address of a pack list (https://...) and restart PopTracker.\n"
                "Remote packs will then be available through the \"Remote\" button in the pack list.",
                Dlg::Buttons::OK, Dlg::Icon::Info);
    };

    auto res = Dlg::MsgBox("PopTracker",
            "Do you want to use a remote pack list?\n\n"
            "A remote pack list lets you browse and download packs from inside PopTracker. "
            "PopTracker does not create, host or endorse the packs in any list.\n\n"
            "The community pack list is suggested. Choose Yes to enter a list address "
            "(leave it empty to use the community pack list).",
            Dlg::Buttons::YesNo, Dlg::Icon::Question);
    if (res != Dlg::Result::Yes) {
        declined();
        return;
    }

    std::string url;
    while (true) {
        url.clear();
        if (!Dlg::InputBox("PopTracker",
                "Enter the address of a pack list, or leave empty for the community pack list",
                "", url)) {
            declined(); // dialog cancelled
            return;
        }
        strip(url);
        if (url.empty()) url = COMMUNITY_URL;
        if (_packManager->addRepository(url)) break;
        Dlg::MsgBox("PopTracker", "The address has to start with https://",
                Dlg::Buttons::OK, Dlg::Icon::Error);
    }

    _config["remote_pack_list_url"] = url;
    saveConfig();

    // validate in the background (needs the main loop to run); on failure, drop it and ask again next start
    _packManager->testRepository(url, [this, url](bool valid) {
        if (valid) return;
        _packManager->clearRepositories();
        _config.erase("remote_pack_list_url");
        saveConfig();
        Dlg::MsgBox("PopTracker",
                "Could not load a valid pack list from:\n" + sanitize_print(url) + "\n\n"
                "You will be asked again on the next start, or you can set "
                "\"remote_pack_list_url\" in PopTracker.json.",
                Dlg::Buttons::OK, Dlg::Icon::Error);
    });
}
```

In `start()`, right before `_frameTimer = std::chrono::steady_clock::now();`:

```cpp
if (_config.find("remote_pack_list_url") == _config.end())
    promptRemotePackList();
```

### 3. Downloading a selected remote version

In `start()`, next to the other `_win->on...` connections:

```cpp
_win->onRemotePackVersionSelected += {this, [this](void*, const std::string& uid,
        const std::string& version, const std::string& url, const std::string& sha256) {
    const std::string name = _packManager->getAvailablePackName(uid);
    const std::string msg = "Download " + name + " " + version + " from\n" + url + " ?";
    if (Dlg::MsgBox("PopTracker", msg, Dlg::Buttons::YesNo, Dlg::Icon::Question) != Dlg::Result::Yes)
        return;
    const auto& installDir = getPackInstallDir();
    fs::create_directories(installDir);
    _packManager->downloadUpdate(url, installDir, uid, version, sha256);
}};
```

### 4. Drop the old "install from community list" branch

In the `onUpdateAvailable` handler, the whole `if (!_pack || _pack->getUID() != uid) { ... }`
block (which prompted "Install X from the community pack list?") is no longer reachable from
the UI. Replace it with:

```cpp
if (!_pack || _pack->getUID() != uid)
    return; // remote installs go through onRemotePackVersionSelected
```

`onUpdateComplete` already calls `_win->refreshPackList()` for non-current packs; that now
rescans installed packs, so the pack moves from the remote to the local list by UID.

## Notes / things to check

- `GetFile()` in packmanager.cpp: `if (!f) cb(false, strerror(errno));` is missing a
  `return;` and would continue with a null `FILE*`. Worth fixing while you're in there.
- CLI `InstallPack` / `ListPacks` rely on the repository being added; they now only work if
  `remote_pack_list_url` is configured (reasonable, but worth knowing).
- All C++17-compatible (no designated initializers, no `std::format`).
