#!/usr/bin/env python3
# spectacle-sni.py
#
# KDE Plasma StatusNotifierItem (tray icon) for Spectacle with DBusMenu:
# - Left-click: repeat last used mode; if none, open Spectacle UI
# - Right-click menu:
#     Open Spectacle UI
#     -------------------
#     (radio) Rectangular Region / Active Window / Full Screen  [✓ shows last used]
#     -------------------
#     [✓] Copy to clipboard
#     [✓] Include mouse pointer
#     [✓] Include window decorations
#     -------------------
#     Quit
# - Persists state to: ~/.config/spectacle-sni.json
# Dependencies: python python-pydbus python-gobject libdbusmenu-glib gobject-introspection spectacle

import json
import os
import subprocess
from pathlib import Path
from pydbus import SessionBus

import gi
gi.require_version("Dbusmenu", "0.4")
from gi.repository import GLib, Dbusmenu

SPECTACLE_BIN = "spectacle"
CONFIG_PATH = Path.home() / ".config" / "spectacle-sni.json"

MODE_DEFS = [
    ("Rectangular Region", ["-r"], "region"),
    ("Active Window",      ["-a"], "active_window"),
    ("Full Screen",        ["-f"], "fullscreen"),
]
MODE_BY_KEY = {key: (label, args) for (label, args, key) in MODE_DEFS}
MODE_KEY_BY_ARGS = {tuple(args): key for (label, args, key) in MODE_DEFS}


class State:
    def __init__(self):
        self.copy_to_clipboard: bool = False
        self.include_pointer: bool = False
        self.include_decorations: bool = True
        self.last_mode_key: str | None = None  # "region" | "active_window" | "fullscreen" | None

    def last_mode_label(self) -> str:
        if self.last_mode_key and self.last_mode_key in MODE_BY_KEY:
            return MODE_BY_KEY[self.last_mode_key][0]
        return "None"

    def last_mode_args(self) -> list[str] | None:
        if self.last_mode_key and self.last_mode_key in MODE_BY_KEY:
            return MODE_BY_KEY[self.last_mode_key][1]
        return None


def load_state() -> State:
    st = State()
    try:
        with CONFIG_PATH.open("r", encoding="utf-8") as f:
            data = json.load(f)

        st.copy_to_clipboard = bool(data.get("copy_to_clipboard", st.copy_to_clipboard))
        st.include_pointer = bool(data.get("include_pointer", st.include_pointer))
        st.include_decorations = bool(data.get("include_decorations", st.include_decorations))

        last_mode_key = data.get("last_mode_key", None)
        if last_mode_key in MODE_BY_KEY:
            st.last_mode_key = last_mode_key
    except FileNotFoundError:
        pass
    except Exception:
        pass
    return st


def save_state(st: State) -> None:
    try:
        CONFIG_PATH.parent.mkdir(parents=True, exist_ok=True)
        data = {
            "copy_to_clipboard": st.copy_to_clipboard,
            "include_pointer": st.include_pointer,
            "include_decorations": st.include_decorations,
            "last_mode_key": st.last_mode_key,
        }
        tmp = CONFIG_PATH.with_suffix(".json.tmp")
        with tmp.open("w", encoding="utf-8") as f:
            json.dump(data, f, indent=2)
        tmp.replace(CONFIG_PATH)
    except Exception:
        pass


def open_spectacle_ui():
    subprocess.Popen([SPECTACLE_BIN, "-l"], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)


def run_spectacle_capture(st: State, capture_args: list[str], update_last: bool = True):
    if update_last:
        key = MODE_KEY_BY_ARGS.get(tuple(capture_args))
        if key:
            st.last_mode_key = key
            save_state(st)

    cmd = [SPECTACLE_BIN]

    if st.copy_to_clipboard:
        cmd += ["-b"] + capture_args + ["-c"]
        if st.include_pointer:
            cmd += ["-p"]
        if not st.include_decorations:
            cmd += ["-e"]  # no-decoration
    else:
        cmd += capture_args

    subprocess.Popen(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)


class SpectacleSNI:
    """
    <node>
      <interface name="org.kde.StatusNotifierItem">
        <method name="Activate">
          <arg type="i" name="x" direction="in"/>
          <arg type="i" name="y" direction="in"/>
        </method>

        <property name="Category" type="s" access="read"/>
        <property name="Id" type="s" access="read"/>
        <property name="Title" type="s" access="read"/>
        <property name="Status" type="s" access="read"/>
        <property name="IconName" type="s" access="read"/>
        <property name="ToolTip" type="(sa(iiay)ss)" access="read"/>
        <property name="Menu" type="o" access="read"/>
      </interface>
    </node>
    """

    Category = "ApplicationStatus"
    Id = "spectacle"
    Title = "Spectacle"
    Status = "Active"
    IconName = "spectacle"

    def __init__(self, menu_object_path: str, st: State):
        self._menu_path = menu_object_path
        self._st = st

    @property
    def ToolTip(self):
        desc = f"Left-click: last used mode (or UI)\nLast: {self._st.last_mode_label()}"
        return ("spectacle", [], "Spectacle", desc)

    @property
    def Menu(self):
        return self._menu_path

    def Activate(self, x, y):
        args = self._st.last_mode_args()
        if args:
            run_spectacle_capture(self._st, args, update_last=False)
        else:
            open_spectacle_ui()


def get_watcher(bus: SessionBus):
    for name in ("org.kde.StatusNotifierWatcher", "org.freedesktop.StatusNotifierWatcher"):
        try:
            return bus.get(name, "/StatusNotifierWatcher")
        except Exception:
            pass
    raise RuntimeError("No StatusNotifierWatcher found on the session bus.")


def add_separator(root):
    sep = Dbusmenu.Menuitem.new()
    sep.property_set("type", "separator")
    root.child_append(sep)
    return sep


def add_plain_item(root, label, callback):
    item = Dbusmenu.Menuitem.new()
    item.property_set("label", label)
    item.property_set_bool("enabled", True)

    def activated(_item, _timestamp):
        callback()

    item.connect("item-activated", activated)
    root.child_append(item)
    return item


def add_check_item(root, label, initial: bool, on_toggle):
    item = Dbusmenu.Menuitem.new()
    item.property_set("label", label)
    item.property_set("toggle-type", "checkmark")
    item.property_set_int("toggle-state", 1 if initial else 0)
    item.property_set_bool("enabled", True)

    def activated(_item, _timestamp):
        new_val = item.property_get_int("toggle-state") == 0
        item.property_set_int("toggle-state", 1 if new_val else 0)
        on_toggle(new_val)

    item.connect("item-activated", activated)
    root.child_append(item)
    return item


def add_radio_item(root, label, initial_selected: bool, on_select):
    item = Dbusmenu.Menuitem.new()
    item.property_set("label", label)
    item.property_set("toggle-type", "radio")
    item.property_set_int("toggle-state", 1 if initial_selected else 0)
    item.property_set_bool("enabled", True)

    def activated(_item, _timestamp):
        on_select()

    item.connect("item-activated", activated)
    root.child_append(item)
    return item


def build_menu(menu_path: str, st: State, quit_callback):
    server = Dbusmenu.Server.new(menu_path)

    root = Dbusmenu.Menuitem.new()
    root.property_set("label", "root")

    # New: Open UI
    add_plain_item(root, "Open Spectacle UI", open_spectacle_ui)

    add_separator(root)

    # Radio capture modes (show last used with ✓)
    radio_items: dict[str, Dbusmenu.Menuitem] = {}

    def set_radio_selected(selected_key: str):
        for key, item in radio_items.items():
            item.property_set_int("toggle-state", 1 if key == selected_key else 0)

    for label, args, key in MODE_DEFS:
        initial = (st.last_mode_key == key)

        def make_select_cb(_key=key, _args=args):
            def _cb():
                st.last_mode_key = _key
                save_state(st)
                set_radio_selected(_key)
                run_spectacle_capture(st, _args, update_last=False)
            return _cb

        radio_items[key] = add_radio_item(root, label, initial, make_select_cb())

    add_separator(root)

    # Toggles (persisted)
    add_check_item(root, "Copy to clipboard", st.copy_to_clipboard,
                   lambda v: (setattr(st, "copy_to_clipboard", v), save_state(st)))
    add_check_item(root, "Include mouse pointer", st.include_pointer,
                   lambda v: (setattr(st, "include_pointer", v), save_state(st)))
    add_check_item(root, "Include window decorations", st.include_decorations,
                   lambda v: (setattr(st, "include_decorations", v), save_state(st)))

    add_separator(root)

    add_plain_item(root, "Quit", quit_callback)

    server.set_root(root)
    return server


def main():
    bus = SessionBus()

    service_name = f"org.kde.StatusNotifierItem.spectacle.pid{os.getpid()}"
    sni_path = "/StatusNotifierItem"
    menu_path = "/StatusNotifierItem/Menu"

    st = load_state()
    loop = GLib.MainLoop()

    def quit_app():
        loop.quit()

    _menu_server = build_menu(menu_path, st, quit_app)

    item = SpectacleSNI(menu_object_path=menu_path, st=st)
    bus.publish(service_name, (sni_path, item))

    watcher = get_watcher(bus)
    watcher.RegisterStatusNotifierItem(sni_path)

    loop.run()


if __name__ == "__main__":
    main()
