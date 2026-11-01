---
name: raven-desktop-app
description: Build, run, package and publish a native cross-platform desktop GUI app (Linux .deb and AppImage, Windows zip and installer) in C with the Raven GUI library (github.com/oscar-soirey/Raven), using VS Code and CMake. Use when the user wants a desktop app and gives only an idea, a scope and the operating systems it must run on.
---

# Raven desktop app skill

Everything an AI agent needs to turn **an app idea + a scope + target operating systems** into a working,
packaged, GitHub-ready desktop app built on [Raven](https://github.com/oscar-soirey/Raven), a C11 GUI library
(window described in XML, logic in C, drawn with Vulkan, no dependencies to install by hand).

This file was written from a real session: a sample app was scaffolded, built, run, packaged as a `.deb` and an
AppImage on Ubuntu 24.04 and confirmed working by the user. Statements are marked:

- **Verified**: done and observed to work (build, `.deb`, AppImage, asset lookup, pinned download).
- **Documented**: stated by Raven's own README or headers, not exercised here.
- **Untested**: written carefully but never run (Windows packaging, macOS, GitHub Actions, Linux older than Ubuntu 24.04).
  Tell the user which of these you relied on.

How to install this skill: save it as `.claude/skills/raven-desktop-app/SKILL.md` in the project (Claude Code),
or `~/.claude/skills/raven-desktop-app/SKILL.md` for every project. For an agent that has no skills feature, paste
it into `AGENTS.md` or `.github/copilot-instructions.md`.

---

## 1. Operating contract

### Inputs you need from the user

| Input | Example | If missing |
|---|---|---|
| App idea and scope | "a markdown notes app with a file tree and live preview" | Ask once. Nothing else works without it. |
| Target OS | Linux, Windows, or both (macOS: best effort) | Ask once, in the same message as the idea if both are missing. |

Everything else you decide yourself and report afterwards: app name, identifiers, folder, widgets, layout, colours,
version `1.0.0`, language (C with an XML interface). Ask **at most one question** in the whole job. Do not ask the
user to choose between technical options; pick the default in this file.

### What you deliver

1. A project folder that configures, builds and runs from VS Code (F5) with no manual steps beyond installing
   the compiler and CMake.
2. The app implemented to the stated scope.
3. Packaging for each requested OS (section 8).
4. A git repository with a first commit, a `README.md`, and step-by-step publish commands (section 9).
5. A short final report (section 11) that says what was verified, what was not, and what the user must test by eye.

### Fit check (do this first, in your head, then tell the user only if there is a problem)

Raven is a good fit for: tools, editors, dashboards, forms and settings screens, file/project browsers, docked
multi-panel workspaces, node graphs, timelines, colour pickers, text editors with syntax colouring, Markdown
display, dialogs, translations, accessibility, and apps that show GPU content (Vulkan/OpenGL viewports).

Limits to respect (all **documented**):

- Needs a **Vulkan-capable driver at run time**. Virtual machines, WSL without GPU support and GPU-less containers
  cannot show the window. Raven exits with `Cannot start Raven. Is a Vulkan driver installed?`-style errors.
- It **draws its own controls**. It imitates the look of Windows 11, macOS and GNOME (`Rlook_set`) but does not use
  native widgets.
- No networking, database, JSON or HTTP library is included. Add a C library through CMake `FetchContent`
  (for example SQLite or cJSON) when the scope needs it, or tell the user it adds work.
- Web pages inside the UI (`raven_web.h`) work by driving an **already installed** Chromium-based browser.
- Not implemented (README "Not implemented yet"): dragging out of a Raven window to other apps, input methods
  (Chinese/Japanese/Korean) on Wayland, a language server client, multiple cursors and search/replace in the
  text editor, full complex-script shaping.
- Version 0.2.0 and **the API is declared unstable**. Always pin Raven to a commit (section 3).
- Not for mobile or the browser.

If the idea needs something Raven cannot do, say so in one sentence, propose the closest feasible scope, and build
that. Do not stall.

Other languages: Raven has bindings for C++, Python, Rust, Go, C#, Zig and LuaJIT
(`bindings/`, and `examples/languages/hello_<language>` in the Raven repo show the same program in each). This skill's
templates and packaging are for **C**. Use another language only if the user asks, and say that packaging for it is not covered.

---

## 2. Prerequisites on the developer machine

| OS | Install |
|---|---|
| Ubuntu / Debian | `sudo apt install build-essential cmake git libx11-dev ninja-build dpkg-dev` (**verified**) |
| Fedora | `sudo dnf install gcc cmake git libX11-devel ninja-build` (documented) |
| Arch | `sudo pacman -S base-devel cmake git libx11 ninja` (documented) |
| Windows | Visual Studio 2022 with the **Desktop development with C++** workload (includes CMake), and Git. MinGW-w64 via MSYS2 also works (documented) |
| macOS | `xcode-select --install`, `brew install cmake`, and `brew install molten-vk` for Vulkan (documented) |

VS Code extensions: **CMake Tools** (`ms-vscode.cmake-tools`) and **C/C++** (`ms-vscode.cpptools`).
Runtime: a GPU driver with Vulkan (`libvulkan1` on Linux).

Raven needs CMake 3.16 or newer and a C11 compiler. The Vulkan headers are downloaded automatically during configure
(nothing Vulkan is linked). On Linux, Wayland libraries are opened at run time; nothing Wayland is needed to build.

---

## 3. Naming rules and the project layout

Pick **one identifier** and use it everywhere. Mismatches break desktop integration (on Wayland the app id must
match the `.desktop` file).

| Placeholder | Meaning | Example |
|---|---|---|
| `{{APP_ID}}` | lowercase, letters digits underscore. Executable name, `.desktop` file name, icon name, install folder | `notes_app` |
| `{{APP_NAME}}` | display name | `Notes App` |
| `{{APP_PKG}}` | Debian package name: `{{APP_ID}}` with `_` changed to `-` | `notes-app` |
| `{{APP_DESC}}` | one line description | `A markdown notes app` |

Layout (this is what the templates in section 4 create):

```
{{APP_ID}}/
├── CMakeLists.txt             finds or downloads Raven (pinned), builds, installs, packages
├── src/
│   ├── main.c                 start-up and the loop
│   ├── app.c / app.h          actions and the app's logic
│   ├── assets.c / assets.h    finds main.xml and strings.csv once installed
│   └── userdata.c / userdata.h  (optional) settings folder per OS
├── assets/
│   ├── main.xml               the interface
│   └── strings.csv            translations (en, fr, ...)
├── packaging/
│   ├── {{APP_ID}}.desktop     Linux menu entry (needs Icon=)
│   ├── make_icon.py           writes a placeholder 256x256 icon
│   ├── {{APP_ID}}.png         the icon (generated, then replaced by the user's own)
│   ├── package-linux.sh       .deb and AppImage
│   └── package-windows.ps1    zip and optional installer
├── .vscode/ settings.json, launch.json, extensions.json
├── .github/workflows/build.yml  builds Linux and Windows packages on GitHub
├── .gitignore  README.md
```

How Raven is obtained, in this order (all three **verified** except where noted):

1. an installed package (`-DCMAKE_PREFIX_PATH=<Raven install folder>`),
2. a local checkout (`-DRAVEN_SOURCE_DIR=<path>`),
3. downloaded from GitHub with `FetchContent`, **pinned to a commit** (default). The download lands in
   `build/_deps/raven-src` and `build/_deps/raven_vulkan_headers-src`. These are inside `build/`, are git-ignored,
   and are not installed on the system. Deleting `build/` removes them.

The pinned commit in the template, `1d5c735fed18ef054bc47bc08c4cd50d0e63b376`, is the one this skill was tested
with (library version 0.2.0). To update Raven: change the hash, rebuild, fix any API changes, retest.

---

## 4. Template files

Write these files, replacing the `{{...}}` placeholders. They were extracted from this document and built (see
section 12). Keep their structure; add features by adding source files, not by rewriting the build.

<!-- file: CMakeLists.txt -->
```cmake
cmake_minimum_required(VERSION 3.16)
project({{APP_ID}} VERSION 1.0.0 LANGUAGES C)

set(APP_ID "{{APP_ID}}")
set(APP_NAME "{{APP_NAME}}")
set(APP_PKG "{{APP_PKG}}")
set(APP_DESC "{{APP_DESC}}")

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Add every .c file of the app here.
set(APP_SOURCES
	src/main.c
	src/app.c
	src/assets.c
)

# ---------------------------------------------------------------------------
# Where Raven comes from, in this order:
#   1. an installed package        (-DCMAKE_PREFIX_PATH=<Raven install folder>)
#   2. a local checkout of Raven   (-DRAVEN_SOURCE_DIR=<path to the Raven repo>)
#   3. downloaded from GitHub      (the default: nothing to install by hand)
# ---------------------------------------------------------------------------
set(RAVEN_SOURCE_DIR "" CACHE PATH "Path to a local Raven checkout (optional)")

find_package(raven QUIET)

if(NOT raven_FOUND)
	# Only the library: no examples, tests or designer.
	set(RAVEN_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
	set(RAVEN_BUILD_TESTS OFF CACHE BOOL "" FORCE)
	set(RAVEN_BUILD_TOOLS OFF CACHE BOOL "" FORCE)

	if(RAVEN_SOURCE_DIR)
		message(STATUS "Using Raven from ${RAVEN_SOURCE_DIR}")
		add_subdirectory("${RAVEN_SOURCE_DIR}" raven EXCLUDE_FROM_ALL)
	else()
		message(STATUS "Raven not found: downloading it from GitHub")
		include(FetchContent)
		# Pinned to a commit so every build uses the same Raven (its API still
		# changes). To update: replace the hash with a newer commit or tag of
		# https://github.com/oscar-soirey/Raven and test again.
		FetchContent_Declare(raven
			GIT_REPOSITORY https://github.com/oscar-soirey/Raven.git
			GIT_TAG 1d5c735fed18ef054bc47bc08c4cd50d0e63b376)
		FetchContent_MakeAvailable(raven)
	endif()

	# GCC warns about a message buffer inside Raven's own theme.c. It is not in
	# this project and does not affect it, so keep it out of the build log.
	if(TARGET raven AND CMAKE_C_COMPILER_ID STREQUAL "GNU")
		target_compile_options(raven PRIVATE -Wno-format-truncation)
	endif()
endif()

add_executable(${APP_ID} ${APP_SOURCES})
target_link_libraries(${APP_ID} PRIVATE raven::raven)
if(MSVC)
	target_compile_definitions(${APP_ID} PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

# main.xml and strings.csv are read when the program runs, from its working
# folder. VS Code starts it in assets/ (see .vscode/), so saving main.xml while
# the program runs updates the window without building again. An installed
# program looks for them next to the executable (see src/assets.c).

# A release build on Windows should not open a console window behind the GUI.
if(MSVC)
	target_link_options(${APP_ID} PRIVATE
		"$<$<CONFIG:Release>:/SUBSYSTEM:WINDOWS>" "$<$<CONFIG:Release>:/ENTRY:mainCRTStartup>")
elseif(WIN32)
	target_link_options(${APP_ID} PRIVATE "$<$<CONFIG:Release>:-mwindows>")
endif()

# ---------------------------------------------------------------------------
# Packaging: `cpack` makes a .deb on Linux, a .zip (or an installer .exe) on
# Windows. Everything belongs to the component "app", so the files Raven itself
# installs (headers, static library, docs) stay out of the package.
# ---------------------------------------------------------------------------
include(GNUInstallDirs)

if(WIN32)
	set(APP_BIN_DIR ".")
	set(APP_ASSETS_DIR "assets")
else()
	set(APP_BIN_DIR "${CMAKE_INSTALL_BINDIR}")
	set(APP_ASSETS_DIR "${CMAKE_INSTALL_DATADIR}/${APP_ID}")
endif()

install(TARGETS ${APP_ID} RUNTIME DESTINATION "${APP_BIN_DIR}" COMPONENT app)
install(FILES assets/main.xml assets/strings.csv DESTINATION "${APP_ASSETS_DIR}" COMPONENT app)
if(UNIX AND NOT APPLE)
	install(FILES packaging/${APP_ID}.desktop
		DESTINATION "${CMAKE_INSTALL_DATADIR}/applications" COMPONENT app)
	install(FILES packaging/${APP_ID}.png
		DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/256x256/apps" COMPONENT app)
endif()

set(CPACK_PACKAGE_NAME "${APP_PKG}")
set(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")
set(CPACK_PACKAGE_VENDOR "Your Name")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${APP_DESC}")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "${APP_NAME}")
set(CPACK_COMPONENTS_ALL app)
set(CPACK_COMPONENTS_GROUPING ALL_COMPONENTS_IN_ONE)

if(WIN32)
	# zip: unzip and run the .exe.  Installer: cpack -G NSIS (needs NSIS).
	set(CPACK_GENERATOR "ZIP")
	set(CPACK_NSIS_DISPLAY_NAME "${APP_NAME}")
	set(CPACK_NSIS_INSTALLED_ICON_NAME "${APP_ID}.exe")
	set(CPACK_PACKAGE_EXECUTABLES "${APP_ID}" "${APP_NAME}")
else()
	set(CPACK_GENERATOR "DEB")
	set(CPACK_DEB_COMPONENT_INSTALL ON)
	set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Your Name <you@example.com>")
	set(CPACK_DEBIAN_PACKAGE_SECTION "utils")
	# libc and libX11 are found from the binary; Raven loads Vulkan at run time, so say it here.
	set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
	set(CPACK_DEBIAN_PACKAGE_DEPENDS "libvulkan1")
	set(CPACK_DEBIAN_PACKAGE_RECOMMENDS "mesa-vulkan-drivers")
	set(CPACK_DEBIAN_FILE_NAME DEB-DEFAULT)
endif()

include(CPack)
```

<!-- file: src/assets.h -->
```c
#ifndef APP_ASSETS_H
#define APP_ASSETS_H

/*
 * Makes the working folder the one that holds main.xml and strings.csv.
 * Does nothing when main.xml is already in the working folder (development:
 * VS Code starts the program in assets/).
 */
void assets_enter_folder(const char* app_id);

#endif
```

<!-- file: src/assets.c -->
```c
#include "assets.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#ifdef _WIN32
#include <direct.h>
#define change_dir _chdir
#else
#include <unistd.h>
#define change_dir chdir
#endif

static int file_exists(const char* path) {
	FILE* file = fopen(path, "rb");
	if (file) fclose(file);
	return file != NULL;
}

/* Folder of the running program, or an empty string if it is not known. */
static void executable_folder(char* folder, size_t size) {
	folder[0] = '\0';
#ifdef _WIN32
	{
		char* path = NULL;
		if (_get_pgmptr(&path) != 0 || !path) return;
		snprintf(folder, size, "%s", path);
	}
#else
	{
		ssize_t length = readlink("/proc/self/exe", folder, size - 1);
		if (length <= 0) return;
		folder[length] = '\0';
	}
#endif
	{
		char* slash = strrchr(folder, '/');
		char* backslash = strrchr(folder, '\\');
		if (backslash && (!slash || backslash > slash)) slash = backslash;
		if (slash) *slash = '\0';
		else folder[0] = '\0';
	}
}

/*
 * Looked up next to the executable:
 *   <exe folder>/assets                  a Windows install, or a zip
 *   <exe folder>/../share/<app_id>       a Linux install or an AppImage (/usr/bin + /usr/share)
 * macOS is not handled (no /proc/self/exe): add _NSGetExecutablePath there.
 */
void assets_enter_folder(const char* app_id) {
	char folder[1024], relative[2][160], candidate[1400];
	int i;

	if (file_exists("main.xml")) return;
	executable_folder(folder, sizeof(folder));
	if (!folder[0]) return;
	snprintf(relative[0], sizeof(relative[0]), "assets");
	snprintf(relative[1], sizeof(relative[1]), "../share/%s", app_id);
	for (i = 0; i < 2; i++) {
		snprintf(candidate, sizeof(candidate), "%s/%s/main.xml", folder, relative[i]);
		if (file_exists(candidate)) {
			snprintf(candidate, sizeof(candidate), "%s/%s", folder, relative[i]);
			if (change_dir(candidate) == 0) return;
		}
	}
}
```

<!-- file: src/app.h -->
```c
#ifndef APP_H
#define APP_H

#include <Raven.h>

/* Binds every action named in main.xml (OnClicked="area.verb"). Call before the window is created. */
void app_register_actions(void);

/* Called once the window exists: fill lists, set initial texts, start timers. */
void app_window_ready(Rid window);

#endif
```

<!-- file: src/app.c -->
```c
#include "app.h"

#include <stdio.h>

/* The state of the app lives here, not in the widgets. */
static struct {
	Rid window;
	int clicks;
} app;

/* OnClicked="app.count" */
static void on_count(Rid sender, const char* action, void* user_data) {
	(void)sender;
	(void)action;
	(void)user_data;
	app.clicks++;
	Rlabel_set_text(Rwidget_find(app.window, "counter"), Rtr_plural("clicks", app.clicks));
}

/* OnClicked="app.reset" */
static void on_reset(Rid sender, const char* action, void* user_data) {
	(void)sender;
	(void)action;
	(void)user_data;
	app.clicks = 0;
	Rlabel_set_text(Rwidget_find(app.window, "counter"), Rtr("app.none"));
}

/* OnClicked="app.quit" */
static void on_quit(Rid sender, const char* action, void* user_data) {
	(void)sender;
	(void)action;
	(void)user_data;
	Rquit(0);
}

void app_register_actions(void) {
	Raction_bind("app.count", on_count, NULL);
	Raction_bind("app.reset", on_reset, NULL);
	Raction_bind("app.quit", on_quit, NULL);
}

void app_window_ready(Rid window) {
	app.window = window;
}
```

<!-- file: src/main.c -->
```c
#include <Raven.h>

#include <stdio.h>

#include "app.h"
#include "assets.h"

static void on_error(RError code, RSeverity severity, const char* detail) {
	fprintf(stderr, "[Raven] %s (%s): %s\n", Rerror_to_string(code), Rseverity_to_string(severity), detail);
}

int main(void) {
	Rid window;

	Rerror_set_callback(on_error);
	assets_enter_folder("{{APP_ID}}");

	if (!Rinit(R_BACKEND_VULKAN)) {
		fprintf(stderr, "Cannot start Raven. Is a Vulkan driver installed?\n");
		return 1;
	}

	/* translations: assets/strings.csv, in the language of the system */
	Rlocale_load_csv("strings.csv");
	Rlocale_set_language(Rlocale_get_system_language());

	app_register_actions();

	window = Rwindow_create_from_xml_file("main.xml");
	if (window == R_INVALID_ID) {
		fprintf(stderr, "Cannot load main.xml (the program must run in the assets/ folder).\n");
		Rshutdown();
		return 1;
	}
	app_window_ready(window);

	/* The loop is the application's. A tool sleeps until something happens
	 * (Rwait_frame); a game that redraws continuously uses Rdraw_frame. */
	while (Rwait_frame(-1.0)) {
	}

	Rshutdown();
	return 0;
}
```

<!-- file: assets/main.xml -->
```xml
<!-- The interface. Save this file while the program runs and the window follows. -->
<Window Title="{{APP_NAME}}" Width="640" Height="420" MinWidth="360" MinHeight="240">
	<VBox Spacing="16" Slot.Padding="24">
		<Label Text="@app.title" TextSize="26"/>
		<HBox Spacing="8">
			<Button Text="@app.button" Style="Primary" OnClicked="app.count"/>
			<Button Text="@app.reset" OnClicked="app.reset"/>
			<Label Name="counter" Text="@app.none" Opacity="0.7" Slot.VAlign="Center"/>
		</HBox>
		<Spacer Slot.Fill="1"/>
		<Button Text="@app.quit" OnClicked="app.quit" Slot.HAlign="Left"/>
	</VBox>
</Window>
```

<!-- file: assets/strings.csv -->
```csv
key,en,fr
app.title,{{APP_NAME}},{{APP_NAME}}
app.button,Click me,Cliquez-moi
app.reset,Reset,Réinitialiser
app.none,No clicks yet,Aucun clic
app.quit,Quit,Quitter
clicks.one,{n} click,{n} clic
clicks.other,{n} clicks,{n} clics
```

<!-- file: src/userdata.h -->
```c
#ifndef APP_USERDATA_H
#define APP_USERDATA_H

#include <stddef.h>

/*
 * Folder where the app keeps its settings and documents, created if needed.
 *   Linux:   $XDG_CONFIG_HOME/<app_id>  or  ~/.config/<app_id>
 *   Windows: %APPDATA%\<app_id>
 * Returns 1 and fills `path` (no trailing slash), or 0. Raven has no such
 * function: use this with Rfile_read / Rfile_write. Add src/userdata.c to
 * APP_SOURCES in CMakeLists.txt when the app saves anything.
 */
int userdata_folder(const char* app_id, char* path, size_t size);

#endif
```

<!-- file: src/userdata.c -->
```c
#include "userdata.h"

#include <stdio.h>
#include <stdlib.h>

#ifdef _WIN32
#include <direct.h>
#define make_dir(p) _mkdir(p)
#else
#include <sys/stat.h>
#include <sys/types.h>
#define make_dir(p) mkdir((p), 0755)
#endif

int userdata_folder(const char* app_id, char* path, size_t size) {
	char base[1024];
#ifdef _WIN32
	const char* appdata = getenv("APPDATA");
	if (!appdata || !appdata[0]) return 0;
	snprintf(base, sizeof(base), "%s", appdata);
#else
	const char* config = getenv("XDG_CONFIG_HOME");
	const char* home = getenv("HOME");
	if (config && config[0]) snprintf(base, sizeof(base), "%s", config);
	else if (home && home[0]) {
		snprintf(base, sizeof(base), "%s/.config", home);
		make_dir(base);
	} else return 0;
#endif
	snprintf(path, size, "%s/%s", base, app_id);
	make_dir(path); /* already existing is fine */
	return 1;
}
```

<!-- file: packaging/{{APP_ID}}.desktop -->
```ini
[Desktop Entry]
Type=Application
Name={{APP_NAME}}
Comment={{APP_DESC}}
Exec={{APP_ID}}
Icon={{APP_ID}}
Terminal=false
Categories=Utility;
```

<!-- file: packaging/make_icon.py -->
```python
#!/usr/bin/env python3
"""Writes a placeholder 256x256 icon: python3 packaging/make_icon.py packaging/{{APP_ID}}.png
The user should replace it with their own 256x256 PNG."""
import math, struct, sys, zlib

W = H = 256
rows = []
for y in range(H):
    row = bytearray([0])
    for x in range(W):
        r = 48
        cx = min(max(x, r), W - 1 - r)
        cy = min(max(y, r), H - 1 - r)
        if (x - cx) ** 2 + (y - cy) ** 2 > r * r:
            row += bytes((0, 0, 0, 0))
            continue
        t = y / (H - 1)
        R, G, B = int(40 + 30 * t), int(60 + 50 * t), int(120 + 60 * t)
        d = math.hypot(x - 128, y - 128)
        if 36 <= d < 70:
            R, G, B = 240, 244, 250
        row += bytes((R, G, B, 255))
    rows.append(bytes(row))

def chunk(tag, data):
    c = struct.pack(">I", len(data)) + tag + data
    return c + struct.pack(">I", zlib.crc32(tag + data) & 0xFFFFFFFF)

png = (b"\x89PNG\r\n\x1a\n"
       + chunk(b"IHDR", struct.pack(">IIBBBBB", W, H, 8, 6, 0, 0, 0))
       + chunk(b"IDAT", zlib.compress(b"".join(rows), 9))
       + chunk(b"IEND", b""))
open(sys.argv[1] if len(sys.argv) > 1 else "icon.png", "wb").write(png)
```

<!-- file: packaging/package-linux.sh -->
```bash
#!/usr/bin/env bash
# Builds Linux packages into dist/.
#   packaging/package-linux.sh            .deb and AppImage
#   packaging/package-linux.sh deb        only the .deb
#   packaging/package-linux.sh appimage   only the AppImage
# Build on the OLDEST distribution you want to support: the packages need at
# least the glibc of the machine they were built on.
set -euo pipefail
cd "$(dirname "$0")/.."

APP_ID="{{APP_ID}}"
WHAT="${1:-all}"
BUILD=build-release
DIST="$PWD/dist"
TOOLS="$PWD/tools"
GEN=()
command -v ninja >/dev/null && GEN=(-G Ninja)

# Extra CMake arguments, e.g. EXTRA_CMAKE_ARGS="-DRAVEN_SOURCE_DIR=/path/to/Raven" to avoid the download.
cmake -S . -B "$BUILD" "${GEN[@]}" -DCMAKE_BUILD_TYPE=Release ${EXTRA_CMAKE_ARGS:-}
cmake --build "$BUILD" --parallel
VERSION="$(sed -n 's/^CMAKE_PROJECT_VERSION:STATIC=//p' "$BUILD/CMakeCache.txt")"
mkdir -p "$DIST"

if [ "$WHAT" = deb ] || [ "$WHAT" = all ]; then
	(cd "$BUILD" && cpack -G DEB)
	cp "$BUILD"/*.deb "$DIST"/
fi

if [ "$WHAT" = appimage ] || [ "$WHAT" = all ]; then
	# AppImages are run without FUSE here, so nothing has to be installed for the build.
	export APPIMAGE_EXTRACT_AND_RUN=1 ARCH=x86_64 LINUXDEPLOY_OUTPUT_VERSION="$VERSION"
	mkdir -p "$TOOLS"
	BASE=https://github.com/linuxdeploy
	[ -f "$TOOLS/linuxdeploy-x86_64.AppImage" ] || \
		curl -fsSL -o "$TOOLS/linuxdeploy-x86_64.AppImage" "$BASE/linuxdeploy/releases/download/continuous/linuxdeploy-x86_64.AppImage"
	[ -f "$TOOLS/linuxdeploy-plugin-appimage-x86_64.AppImage" ] || \
		curl -fsSL -o "$TOOLS/linuxdeploy-plugin-appimage-x86_64.AppImage" "$BASE/linuxdeploy-plugin-appimage/releases/download/continuous/linuxdeploy-plugin-appimage-x86_64.AppImage"
	chmod +x "$TOOLS"/*.AppImage
	ln -sf linuxdeploy-plugin-appimage-x86_64.AppImage "$TOOLS/linuxdeploy-plugin-appimage"
	export PATH="$TOOLS:$PATH"

	rm -rf AppDir
	cmake --install "$BUILD" --prefix AppDir/usr --component app
	(cd "$DIST" && "$TOOLS/linuxdeploy-x86_64.AppImage" --appdir ../AppDir \
		-d ../AppDir/usr/share/applications/$APP_ID.desktop \
		-i ../AppDir/usr/share/icons/hicolor/256x256/apps/$APP_ID.png \
		--output appimage)
fi

echo "Done. Files in $DIST:"
ls -l "$DIST"
```

<!-- file: packaging/package-windows.ps1 -->
```powershell
# Builds the Windows package. Run in "Developer PowerShell for VS 2022" from the project folder:
#   powershell -ExecutionPolicy Bypass -File packaging\package-windows.ps1
# The zip holds the .exe and its assets folder. For a setup .exe, install NSIS
# (https://nsis.sourceforge.io) and pass -Installer.
param([switch]$Installer)
$ErrorActionPreference = "Stop"
Set-Location (Join-Path $PSScriptRoot "..")

cmake -S . -B build
cmake --build build --config Release --parallel
Push-Location build
cpack -C Release
if ($Installer) { cpack -C Release -G NSIS }
Pop-Location

New-Item -ItemType Directory -Force dist | Out-Null
Copy-Item build\*.zip dist\ -Force
if ($Installer) { Copy-Item build\*.exe dist\ -Force -ErrorAction SilentlyContinue }
Get-ChildItem dist
```

<!-- file: .vscode/settings.json -->
```json
{
	"cmake.configureOnOpen": true,
	"cmake.buildDirectory": "${workspaceFolder}/build",
	// Run from assets/ so main.xml and strings.csv are found, and so saving
	// main.xml while the program runs updates the window.
	"cmake.debugConfig": {
		"cwd": "${workspaceFolder}/assets"
	},
	"cmake.launchCwd": "${workspaceFolder}/assets",
	// Optional: use a local Raven checkout instead of downloading it.
	// "cmake.configureSettings": { "RAVEN_SOURCE_DIR": "C:/path/to/Raven" },
	"files.associations": {
		"*.xml": "xml"
	}
}
```

<!-- file: .vscode/launch.json -->
```json
{
	"version": "0.2.0",
	"configurations": [
		{
			"name": "Debug (gdb / lldb: Linux, macOS, MinGW)",
			"type": "cppdbg",
			"request": "launch",
			"program": "${command:cmake.launchTargetPath}",
			"args": [],
			"cwd": "${workspaceFolder}/assets",
			"stopAtEntry": false,
			"MIMode": "gdb",
			"setupCommands": [
				{ "description": "Pretty printing", "text": "-enable-pretty-printing", "ignoreFailures": true }
			],
			"preLaunchTask": "CMake: build"
		},
		{
			"name": "Debug (Visual Studio debugger: Windows MSVC)",
			"type": "cppvsdbg",
			"request": "launch",
			"program": "${command:cmake.launchTargetPath}",
			"args": [],
			"cwd": "${workspaceFolder}/assets",
			"stopAtEntry": false,
			"console": "integratedTerminal",
			"preLaunchTask": "CMake: build"
		}
	]
}
```

<!-- file: .vscode/extensions.json -->
```json
{
	"recommendations": [
		"ms-vscode.cmake-tools",
		"ms-vscode.cpptools"
	]
}
```

<!-- file: .gitignore -->
```gitignore
build/
build-release/
.cache/
compile_commands.json
tools/
AppDir/
dist/
*.AppImage
*.deb
```

<!-- file: .github/workflows/build.yml -->
```yaml
# Untested template. Builds the Linux and Windows packages on GitHub, so nobody
# needs a Windows machine. Push a tag (git tag v1.0.0 && git push origin v1.0.0)
# to attach the files to a GitHub Release.
name: Build

on:
  push:
    branches: ["**"]
    tags: ["v*"]
  pull_request:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  linux:
    # the oldest runner, so the packages work on older distributions too
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - name: Dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends libx11-dev ninja-build dpkg-dev
      - name: Packages
        run: bash packaging/package-linux.sh all
      - uses: actions/upload-artifact@v4
        with:
          name: linux
          path: dist/*

  windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and package
        shell: pwsh
        run: |
          cmake -S . -B build
          cmake --build build --config Release --parallel
          cd build
          cpack -C Release
      - uses: actions/upload-artifact@v4
        with:
          name: windows
          path: build/*.zip

  release:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: [linux, windows]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: artifacts
      - uses: softprops/action-gh-release@v2
        with:
          files: artifacts/**/*
```

<!-- file: README.md -->
```markdown
# {{APP_NAME}}

{{APP_DESC}}. Built with [Raven](https://github.com/oscar-soirey/Raven) (C, XML interface, Vulkan).

## Run in VS Code

1. Install a C compiler and CMake (Windows: Visual Studio 2022 with the C++ workload; Ubuntu:
   `sudo apt install build-essential cmake git libx11-dev ninja-build`).
2. Install the CMake Tools and C/C++ extensions, open this folder, pick a kit, press F5.
   The first build downloads and builds Raven (a few minutes).
3. A Vulkan driver is required at run time.

The program runs in `assets/`: saving `main.xml` while it runs updates the window.

## Packages

- Linux: `bash packaging/package-linux.sh` writes a `.deb` and an AppImage to `dist/`.
- Windows: `powershell -ExecutionPolicy Bypass -File packaging\package-windows.ps1` writes a zip to `dist\`.
- Both: push a tag `v1.0.0`; GitHub Actions builds both and attaches them to a release.

Before publishing, change the vendor and maintainer in `CMakeLists.txt` and replace `packaging/{{APP_ID}}.png`.
```

---

## 5. Raven API cheat sheet

Everything is `#include <Raven.h>`. Every object is an opaque `Rid` handle (validated on each call; a deleted
object is reported through the error callback, not a crash). Functions are `R<object>_<action>`; constants
`R_<NAME>`; types `R<Name>`. **Everything must be called from the UI thread** (the one that called `Rinit`) except
`Rtask_post`, `Rtask_run_async`, `Rwake_up`, `Rquit`, `Rwindow_request_redraw`, `Rget_time`.

### Lifecycle (the only structure an app needs)

```c
Rerror_set_callback(on_error);                 /* first: see every warning, including XML mistakes */
Rinit(R_BACKEND_VULKAN);                       /* 0 on failure */
Rlocale_load_csv("strings.csv");
Rlocale_set_language(Rlocale_get_system_language());
Raction_bind("area.verb", handler, user_data); /* before the window, so XML can name it */
Rid window = Rwindow_create_from_xml_file("main.xml");
while (Rwait_frame(-1.0)) {}                   /* sleeps until something happens; Rdraw_frame never waits */
Rshutdown();                                   /* frees every window, widget and resource */
```

### The pattern: XML describes, C reacts

1. In `main.xml`, give widgets a `Name`, and name an action on the widgets that fire one
   (`OnClicked`, `OnChanged`, `OnCommitted`, `OnActivated` are synonyms of `Action`).
2. In C, `Raction_bind("name", fn, user_data)`; the handler is `void fn(Rid sender, const char* action, void* user_data)`.
   `sender` is the widget that fired it, so read its value from there.
3. Find any widget with `Rwidget_find(window, "name")` and change it with the C API, or set one XML attribute
   at run time with `Rwidget_set_attribute(widget, "Text", "Hello")`.

Name actions `area.verb` (`file.save`, `edit.undo`, `note.delete`). One handler can serve many widgets: look at `action` or `sender`.

When the action fires:

| Widget | Fires when |
|---|---|
| Button, MenuItem | clicked |
| Checkbox, Slider, SpinBox, ComboBox, TabView, Expander | the value changes |
| TextInput | Enter is pressed (every keystroke: `Rtext_input_set_changed_callback`) |
| ListView, TreeView | a row is activated (double click or Enter) |

### Reading and writing widget values

| Widget | Read | Write |
|---|---|---|
| Label | `Rlabel_get_text` | `Rlabel_set_text` |
| TextInput | `Rtext_input_get_text` | `Rtext_input_set_text` |
| TextEditor | `Rtext_editor_get_text` | `Rtext_editor_set_text`, `Rtext_editor_set_language(editor, Rlanguage_for_file(path))` |
| Checkbox | `Rcheckbox_is_checked` | `Rcheckbox_set_checked` |
| Slider | `Rslider_get_value` | `Rslider_set_value`, `Rslider_set_range` |
| SpinBox | `Rspin_box_get_value` | `Rspin_box_set_value` |
| ProgressBar | `Rprogress_bar_get_value` (0..1) | `Rprogress_bar_set_value` |
| ComboBox | `Rcombo_box_get_selected` | `Rcombo_box_add_item`, `Rcombo_box_clear_items`, `Rcombo_box_set_selected` |
| ListView | `Rlist_view_get_selected` | `Rlist_view_add_item`, `Rlist_view_clear_items`, `Rlist_view_set_selected`; long lists: `Rlist_view_set_virtual_items(list, count, text_callback, user_data)` |
| any widget | | `Rwidget_set_visibility`, `Rwidget_set_enabled`, `Rwidget_set_tooltip`, `Rwindow_set_title(window, text)` |

Build widgets from C when the content is dynamic: `Rwidget_create(R_WIDGET_VBOX)`, `Rlabel_create("text")`,
`Rbutton_create("text")`, `Rwidget_add_child(parent, child)` (returns the child). Deleting a widget deletes its
children.

### XML essentials (full reference: `docs/XML.md` in the Raven repo)

- Tags are widget types; attributes are properties; case does not matter; text inside an element is its `Text`.
  `<` and `&` must be written `&lt;` and `&amp;` inside attribute values. Unknown tags or attributes only warn
  (through the error callback, with line and column) and loading continues. Look there first when a widget is missing.
- **Panels:** `VBox`, `HBox`, `Grid` (`Columns="auto,1,2"`, `Slot.Column`, `Slot.Row`), `UniformGrid`, `Overlay`,
  `Border` (`Background`, `CornerRadius`, `Shadow`), `SizeBox`, `Spacer`, `ScrollBox`, `Splitter`, `WrapBox`, `Canvas`,
  `Switcher`, `TabView` (`Slot.Title`), `Expander`, `DockArea` (`Slot.Dock`, `Slot.DockTo`), `HeaderBar`, `Island`.
- **Controls:** `Label`, `Button` (`Style="Primary|Flat"`), `Checkbox` (`Style="Check|Radio|Switch"`), `Slider`, `ProgressBar`,
  `SpinBox`, `TextInput`, `ComboBox` (`Items="A|B|C"`), `ListView`, `TreeView`, `Image`, `Separator`, `MenuBar/Menu/MenuItem`
  (`Shortcut="Ctrl+S"`), `TextEditor` (`Language`, `LineNumbers`, `Wrap`), `FileBrowser`, `Viewport`, `Custom`,
  `ColorPicker`, `Timeline`, `NodeGraph`.
- **Layout is Slate-style:** the parent arranges children, and each child carries a **slot**:
  `Slot.Padding`, `Slot.HAlign` (`Fill|Left|Center|Right`), `Slot.VAlign`, `Slot.Fill="1"` (share of the free space),
  `Slot.Auto`. A `HBox` child with `Slot.Fill="1"` takes the rest of the row. Sizes are logical units (scaled per monitor).
- Window attributes: `Title Width Height MinWidth MinHeight Resizable Maximized CustomHeader Transparent Theme`.
  `<Shortcut Keys="Ctrl+S" Action="file.save"/>` as a child of `<Window>` fires an action app-wide.
- Per-widget: `Name Width Height MinWidth Visibility(Visible|Hidden|Collapsed) Enabled Opacity Tooltip Class Material Appear`.
- Translations: `Text="@key"` follows the language live. Plurals: `Rtr_plural("files", n)` with `files.one`/`files.other` rows
  using `{n}`; arguments: `Rtr_format("greeting", args, count)` with `{0}`.
- Reusable pieces: `<Widget Name="Field" Label="Name">...<Children/></Widget>`, then `<Field Label="Email"/>`. Also `If`, `Repeat`,
  `ForEach`, `Include`. Declare shared tags in their own file and load with `Rxml_register_widgets_from_file(path)`.
- **Live reload:** an XML file loaded from disk is watched; saving it updates the open window and keeps handles, callbacks,
  text typed and scroll. A file with a mistake changes nothing and reports the error. `Rxml_set_reload_callback` tells when
  to look widgets up again.

### Dialogs, files, clipboard

```c
RFileFilter filters[] = { { "Markdown", "*.md" }, { "All files", "*" } };
RFileDialogDesc desc = { 0 };
desc.parent = window; desc.title = "Open"; desc.filters = filters; desc.filter_count = 2;
const char* path = Rfile_dialog_open(&desc);          /* NULL when cancelled; Rfile_dialog_save, Rfile_dialog_select_folder */
size_t size; char* text = Rfile_read(path, &size);    /* NUL-terminated; release with Rmemory_free(text) */
Rfile_write(path, data, size);                        /* UTF-8 paths on every platform */
if (Rmessage_box(window, "Quit", "Discard the changes?", R_MESSAGE_OK_CANCEL) == R_MESSAGE_RESULT_OK) { /* ... */ }
Rclipboard_set_text("..."); const char* s = Rclipboard_get_text();
```

File dialogs are the system's (Windows common dialogs; on Linux `zenity` or `kdialog` when installed), otherwise Raven's own
`FileBrowser` dialog. Ask before closing with unsaved work: `Rwindow_set_close_callback(window, fn, data)` where
`int fn(Rid window, void* data)` returns `R_TRUE` to let it close, `R_FALSE` to keep it open.
Files dropped from other apps: `Rwindow_set_drop_callback(window, fn, data)` (`event->paths`, `event->path_count`).
Where to keep settings: `userdata_folder()` from `src/userdata.c` above (Raven has no such function).

### Threads, tasks, timers

```c
Rtask_run_async(work_fn, data);     /* runs void work_fn(void*) on a worker thread (any thread may call) */
Rtask_post(done_fn, data);          /* runs void done_fn(void*) on the UI thread: the ONLY way to touch the UI from a worker */
Rid t = Rtimer_create(1.0, R_TRUE, on_tick, data);   /* every second, repeating; Rtimer_destroy(t) */
```

Never call widget functions from a worker thread. Do the slow work on a worker, then `Rtask_post` the result back.
Keep the app's data in plain C structs and make the worker operate on a copy.

### Look, theme, language

```c
Rlook_set(R_LOOK_NATIVE_DARK);           /* or R_LOOK_NATIVE_LIGHT, R_LOOK_RAVEN_DARK/LIGHT, R_LOOK_WINDOWS_*, R_LOOK_MACOS_*, R_LOOK_GNOME_* */
Rtheme_set_accent(0.13f, 0.83f, 0.93f);  /* one colour and its shades */
Rtheme_load("themes/ocean.xml"); Rtheme_use("Ocean");   /* a theme file */
Raccessibility_follow_system();          /* text size, contrast, motion of the OS; screen readers work with no extra code */
```

Give interactive icon-only buttons `AccessibleName="..."`. Set `RAVEN_WINDOW_SYSTEM=x11` or `wayland` in the environment to force a window system when debugging.

### Bigger features (read the matching example in the Raven repo before writing)

| Need | Raven example folder |
|---|---|
| menus, splitters, tree, tabs, property grid, virtual list, custom widget | `examples/editor` |
| text editor with syntax colours, custom title bar, file dialogs, drop | `examples/notepad` |
| docking panels, file browser, save/restore layout | `examples/workspace` |
| declared tags, `Repeat`, `ForEach`, `Include` | `examples/components` |
| node graph, timeline, colour picker | `examples/studio` |
| themes, looks, Markdown | `examples/themes` |
| 3D or GPU content inside the UI | `examples/viewport_vulkan`, `viewport_opengl` |
| the same app in C++/Rust/Python/Go/C#/Zig/Lua | `examples/languages` |

Raven also ships `raven_designer`, a visual XML designer, in its own build (`RAVEN_BUILD_TOOLS=ON`); optional.

---

## 6. How to structure a real app

- **Keep logic out of widgets.** State lives in C structs in `app.c` (or more files). Handlers read the sender, update
  state, then update widgets in one `refresh_*()` function. This keeps the UI easy to change in XML.
- **One file per area** once `app.c` passes ~400 lines (`notes.c`, `export.c`...). List each in `APP_SOURCES`.
- **Pure logic in its own files with no Raven include** (parsing, calculations, storage). These can get unit tests with
  CTest and run on machines with no GPU and in CI. A Raven window cannot be opened without Vulkan, so never rely on
  opening the UI in an automated test.
- **Layout first in XML**, then bind. A new screen is a new `VBox` inside a `Switcher` or `TabView`, not a new window,
  unless the user wants separate windows.
- **Reusable UI** (a labelled field, a card, a toolbar button) goes into `<Widget>` declarations.
- **Never block the UI thread** (file scans, network, heavy compute): use `Rtask_run_async` + `Rtask_post`.
- **Strings go in `strings.csv`** from the start (`@key` in XML, `Rtr` in C). Adding a language later is then one column.
- **Free what you `Rfile_read`** with `Rmemory_free`. Everything else Raven frees at `Rshutdown`.
- **Handle every startup failure** with a message on stderr, as the templates do.
- Keep the project in C11. Declare variables at the top of blocks in files that must build with MSVC's C mode if unsure;
  avoid GCC-only extensions (statement expressions, `typeof`, variable-length arrays).
- Do not use Linux-only calls (`unistd.h`, `/proc`) outside `#ifdef`s. Windows paths use `\`, but `/` works in C file functions.

---

## 7. Workflow (run these phases in order)

Create a task list with these phases and tick them off.

1. **Understand.** Reduce the idea to: screens, data the app stores, actions the user takes, files it reads or writes. Write
   it as 5 to 15 bullets in the README under "Scope". Do the fit check (section 1).
2. **Scaffold.** Make the folder `{{APP_ID}}/`, write every template of section 4 with placeholders replaced, run
   `python3 packaging/make_icon.py packaging/{{APP_ID}}.png`, `chmod +x packaging/package-linux.sh`.
3. **Build.** Use a scratch build directory so the user's `build/` stays clean:
   `cmake -S . -B build-check -G Ninja -DCMAKE_BUILD_TYPE=Debug && cmake --build build-check --parallel`.
   On Linux, if `/home/.../raven` or another Raven checkout already exists, add `-DRAVEN_SOURCE_DIR=<path>` to avoid a download.
   The first build compiles Raven (about 1 to 2 minutes with 8 cores); later builds take seconds. Fix every warning you caused.
4. **Implement the scope** in small steps: interface in `main.xml`, strings, then actions in C. After each step rebuild.
   Add `-Wall -Wextra` while developing: `-DCMAKE_C_FLAGS="-Wall -Wextra"` in the scratch build.
5. **Verify** (section 10). You usually cannot open the window (no GPU in agent sandboxes). Say so; do the checks that are possible.
6. **Package** for the requested OS (section 8) and test the package contents.
7. **Git and publish instructions** (section 9).
8. **Report** (section 11).

Run `rm -rf build-check` when done if it was only for checking. Never commit `build/`.

---

## 8. Packaging

Packages must be built **on the target OS** (or by GitHub Actions, section 9). Cross-compiling is not covered.

| Target | Command | Result | Status |
|---|---|---|---|
| Debian, Ubuntu, Mint (amd64) | `bash packaging/package-linux.sh deb` | `dist/<pkg>_1.0.0_amd64.deb` | **Verified** |
| Any Linux, one file | `bash packaging/package-linux.sh appimage` | `dist/<Name>-1.0.0-x86_64.AppImage` | **Verified** |
| Windows, zip | `powershell -File packaging\package-windows.ps1` | `dist\<pkg>-1.0.0-win64.zip` | Untested |
| Windows, setup `.exe` | same with `-Installer` (needs NSIS) | `dist\*.exe` | Untested |
| macOS | not covered (no `.app` bundle rules) | build and run only | Untested |

Linux details (**verified**):

- The `.deb` installs `/usr/bin/<app_id>`, `/usr/share/<app_id>/main.xml`, the `.desktop` file and the icon. `dpkg-shlibdeps`
  finds `libc6` and `libx11-6`; `libvulkan1` is added by hand because Raven loads Vulkan at run time; `mesa-vulkan-drivers` is recommended.
- Only the `app` component is packaged; Raven's own install files (headers, `libraven.a`, docs) are excluded. Do not remove
  `COMPONENT app` from the install rules.
- Install with `sudo apt install ./file.deb` (installs missing dependencies automatically; `dpkg -i` does not).
  The note "Download is performed unsandboxed as root ... `_apt`" appears when the file is inside a home folder; it is harmless.
- The AppImage bundles the small libraries linuxdeploy finds and deliberately **not** `libX11`, `libxcb` or the Vulkan driver.
  The script sets `APPIMAGE_EXTRACT_AND_RUN=1`, so building needs no FUSE. **Running** an AppImage on Ubuntu 22.04+/24.04
  needs FUSE 2: `sudo apt install libfuse2` (Ubuntu 24.04: `libfuse2t64`). Without it, double-clicking does nothing.
- The two `linuxdeploy*.AppImage` files in `tools/` are build tools, not the app. If the user runs them directly, nothing visible happens.
- Building the AppImage needs internet (the tools and `appimagetool`'s runtime are downloaded from GitHub).
- **glibc:** a package needs at least the glibc of the machine it was built on. A build on Ubuntu 24.04 will not install on
  Ubuntu 22.04 or Debian 12. Build on the oldest target (the GitHub workflow uses `ubuntu-22.04` for this reason; untested).
- Distribution names: the `.deb` is for Debian-family only (not Fedora/Arch/openSUSE, not ARM). The AppImage covers those.

Windows details (**documented/untested**):

- Release builds link as a GUI program (no console window). In **Debug** a console stays, which is useful for stderr.
  If the user later wants an exe icon, add a `.rc` resource file; the templates do not.
- The zip holds `<app_id>.exe` and `assets\`. The exe finds `assets\` next to itself (`src/assets.c`). Shipping only the exe will not work.
- Multi-config generators (Visual Studio) need `--config Release` for the build and `cpack -C Release`.
- A Vulkan driver comes with normal GPU drivers; very old or virtual machines may lack it.
- MinGW libraries and MSVC libraries are not interchangeable; stay with one compiler per build.

Updating a version: change `project(... VERSION x.y.z ...)` in `CMakeLists.txt`; package names and AppImage names follow.
Before first publication tell the user to replace `Your Name` and `you@example.com` in `CMakeLists.txt` and the placeholder icon.

---

## 9. Git, GitHub and releases

Commands for the user (Ubuntu; same elsewhere). Do not run `git push` yourself unless the user asks and credentials exist.

```sh
cd <project folder>
git config --global user.name "Your Name"          # skip if already set
git config --global user.email "you@example.com"
git init -b main
git add .
git status          # check there is no build/, dist/, tools/ or AppDir/
git commit -m "Initial commit"
```

Then one of:

```sh
sudo apt install gh && gh auth login               # once
gh repo create <name> --private --source=. --push  # --public for a public repo
```

or create an empty repo on github.com/new (no README), then
`git remote add origin https://github.com/<user>/<name>.git && git push -u origin main`
(GitHub asks for a personal access token as the password, not the account password).

Remind the user to add a `LICENSE` of their choice; Raven is Apache-2.0, which allows commercial use, and what you build with it is yours.
If Raven source is ever redistributed or modified, keep its license and notices.

Releases: attach the `.deb`, AppImage and zip to a GitHub Release
(`gh release create v1.0.0 dist/* --notes "First release"`), or push a `v*` tag and let `.github/workflows/build.yml`
build both OS packages and attach them (**untested** workflow: if a job fails, read the log, fix the step, push again).

The `build/_deps/raven-src` and `raven_vulkan_headers-src` folders that appear on first build are downloaded dependencies
inside `build/`; they are ignored by git and anyone cloning the repo downloads them again on their first build.

---

## 10. Verification (what an agent can and cannot check)

Do all of these that the environment allows and **state the result of each** in the final report.

| Check | How | Typical result in an agent sandbox |
|---|---|---|
| Configure and build with no warnings from the app's files | `cmake` + `cmake --build` on a scratch dir | Possible |
| Raven API names exist | grep each `R...(` used against `include/raven/*.h` of the pinned commit | Possible |
| XML loads without warnings | cannot without a window | Not possible without Vulkan |
| Window appears and behaves | needs Vulkan + a display | **Not possible** in GPU-less sandboxes. A run ends with `Raven: ... no surface extension` or `cannot open the display`, which proves start-up reached `Rinit` only |
| Asset lookup after install | extract the `.deb` (`dpkg-deb -x`), run the binary from `/` under `strace -f -e trace=chdir`: it must `chdir` into `.../share/<app_id>` | Possible (**verified** technique) |
| `.deb` contents and dependencies | `dpkg-deb -I` and `dpkg-deb -c`: only app files, `Depends: libvulkan1, libc6, libx11-6` | Possible |
| AppImage builds | run `package-linux.sh appimage` | Possible with internet |
| Pure-logic unit tests | CTest on the Raven-free files | Possible |

If the window cannot be shown, never claim the UI "works". Say: "builds and starts; please run it and check these screens: ...".
List 3 to 6 concrete things for the user to try (click X, type Y, resize, switch language, close with unsaved data).

Documented debugging aids: set `RAVEN_LIVE_RELOAD=0` to turn off XML watching; `Rerror_set_callback` output shows XML warnings
with line and column; `RFrameStats` and `Rmemory_get_stats` exist for profiling; GPU loss is recovered automatically.

---

## 11. Final report to the user (keep it short)

1. What was built (one paragraph), and the folder.
2. How to run it in VS Code (3 lines) and which OS packages were produced or are ready to produce.
3. **Verified vs not verified** (use the three labels from the top).
4. What to try by eye (the 3 to 6 items).
5. What they must change before publishing (maintainer, vendor, icon, license) and the git commands.
6. Anything from the limits list that affected the scope.

Respect the user's working style: they direct the agent to write the code and then review and test it. Give changes as code or file
paths, not as full project archives, unless asked.

---

## 12. Lessons learned (symptom, cause, fix)

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot start Raven. Is a Vulkan driver installed?` | no Vulkan driver, VM, WSL, or no display | install GPU driver / `mesa-vulkan-drivers`; run on a real desktop session |
| Window never appears in an agent sandbox | no GPU | cannot be fixed there; hand over test steps |
| `X11: cannot open the display` | no `DISPLAY` | run in a desktop session |
| `Cannot load main.xml` | wrong working folder | VS Code `cwd` must be `assets/`; installed apps use `assets.c` |
| Compiler warning `-Wformat-truncation` in Raven's `theme.c` | upstream code, not the app | silenced for Raven's target in the template |
| Build log shows a long GitHub download on first configure | `FetchContent` of Raven and Vulkan headers | normal, once; lives in `build/_deps` |
| Raven changed and the app no longer builds | API is unstable and `main` moved | keep the pinned commit; update deliberately |
| `.deb` contains Raven headers and `libraven.a` | install rules without `COMPONENT app` | keep `COMPONENT app` and `CPACK_COMPONENTS_ALL app` |
| `apt` says `_apt` could not access the file | `.deb` in the home folder | harmless; or copy it to `/tmp` first |
| AppImage double-click does nothing | no FUSE 2 | `sudo apt install libfuse2` (`libfuse2t64` on Ubuntu 24.04) |
| User ran `linuxdeploy-x86_64.AppImage` and nothing happened | those are build tools | run `package-linux.sh appimage`; the app AppImage is in `dist/` |
| `.deb` will not install on older Ubuntu/Debian | glibc newer than the target | build on the oldest target (CI `ubuntu-22.04`) |
| AppImage build fails `Icon` / `.desktop` error | no `Icon=` line or missing PNG | keep `Icon={{APP_ID}}` and the 256x256 PNG |
| App not matched to its menu entry / wrong icon on Wayland | app id differs from `.desktop` name | executable name = `.desktop` base name = icon name = `{{APP_ID}}`; `RAVEN_APP_ID` overrides |
| High CPU while idle | main loop uses `Rdraw_frame` | use `while (Rwait_frame(-1.0)) {}` for tools |
| Windows: exe runs but window text is empty | `assets\` not next to the exe | ship the zip, not the bare exe |
| Windows: console window behind the app | Debug build, or link flags missing | Release build with the flags in the template |
| UI changes from a thread crash or garble | UI functions are not thread-safe | `Rtask_post` back to the UI thread |
| Pasting only the first half of the AppImage commands | the tools are downloaded in step one, packaged in step two | use `package-linux.sh`, which does both |

### What was verified in the original session

- Configure and build against an installed Raven package, against a local checkout (`-DRAVEN_SOURCE_DIR`) and by downloading the
  pinned commit from GitHub (checked out commit equals the pin; zero warnings after the `-Wno-format-truncation` line).
- `.deb` build and install, contents free of Raven headers, dependencies as listed, asset lookup via `/proc/self/exe`.
- AppImage built with `linuxdeploy` + the appimage plugin; the user ran it on Ubuntu 24.04 and it worked. The window was shown
  by the user, never by the agent.
- The `.desktop` entry appears after installing the `.deb`.
- **The templates in section 4 were extracted from this file by script, filled with a test app id and built from scratch**:
  zero warnings from the app's files with `-Wall -Wextra`; `userdata.c` compiled; all 45 Raven function names used in this
  file exist in the pinned headers; `package-linux.sh all` produced a `.deb` (only app files, correct dependencies) and an
  AppImage, and both `chdir` into their `share/<app_id>` folder when started from `/` (checked with `strace`). The window itself
  was not seen by the agent (no GPU).

### Not verified (always disclose when relied upon)

Windows builds and packaging (including the console-hiding link flags and `_get_pgmptr`), NSIS installer, macOS, the GitHub
Actions workflow, builds on Ubuntu 22.04, Wayland sessions, `RAVEN_APP_ID` behaviour, and the `Rwait_frame` main loop used in
the templates (it is the loop of Raven's own `hello` example; the original sample ran with `Rdraw_frame`).
