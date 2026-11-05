# railwl — daily-driver roadmap

Order matters: each tier depends on the one above it.

## Tier 1 — BLOCKING (foundation)
- [x] xdg_popup + xdg_positioner (menus, dropdowns, tooltips, transients)
- [x] popup rendering order (above parent) + click-outside dismiss
- [x] keyboard repeat (rate/delay honored from wl_keyboard.repeat_info)
- [x] no client-side decorations: xdg-decoration forces SERVER_SIDE (no CSD)
- [x] core clipboard: wl_data_device_manager (data_source/offer/device, selection broadcast)
- [x] mouse window move/resize (Super=move, edge-grip=resize) wired in pointer path
- [x] focus follows click (rail_pick_surface: popups > bar > focused tile)
- [x] screen rendering verified: GTK4 app + railbar render to real output, no crash
- [x] client rc_run flush fix (queued commits were never flushed -> deadlock before map)

### Bug fixes (Tier 1)
- [x] blit_view dropped the y offset (tiles drew at row 0) -> respect s->cy
- [x] bar stranded offscreen at cx=output->w -> pin bar to (0,0), skip slide-in
- [x] tile geometry now below bar (y=28, h=output->h-28), not full-height under it
- [x] clipboard teardown type-confusion (dev_res_destroy cast user_data to rail*) -> SIGSEGV
- [x] wl_buffer had NULL impl -> GTK wl_buffer.destroy() aborted; added destroy handler

## Tier 2 — terminal + browser
- [x] stock non-wlroots client: `term` (our own terminal, `~/.local/bin/term`) renders and takes focus; text verified from a captured frame
- [x] clipboard: copy/paste verified end-to-end, regular + primary, in both directions with stock clients (`wl-copy`/`wl-paste`) and with our own reader (`test/clip_test`)
- [ ] Firefox/Chromium: xdg_popup, fullscreen, video, clipboard
- [x] xdg_toplevel configure_bounds / resize edge handling
- [x] mouse cursor rendering (real pointer)

### Fixes landed (Tier 2)
- [x] selection is sent *per protocol*: immediately before keyboard focus, and on change only while focused. It used to be pushed to every device at every opportunity, which handed `wl-paste` an offer during startup; wl-clipboard 2.3.0 frees its throwaway focus surface from inside that callback and then used the dangling pointer -> SIGSEGV. Focus-based delivery is both correct and what the client is written for
- [x] xdg_surface.configure carries a real unique serial (wl_display_next_serial), not a constant 0
- [x] when a mapped surface goes away (destroy or attach(NULL)) the seat focus follows the strip, instead of leaving a stale focus pointer behind
- [x] wl_subsurface implemented for real (was a no-op stub that never created the resource)
- [x] every version-specific event is gated on the version the client bound (wl_keyboard.repeat_info v4, wl_seat.name v2, wl_output.done/name/description, pointer axis_source/axis_discrete, xdg_toplevel.configure_bounds v4) - libwayland-client aborts on an event past the proxy version
- [x] every request on every exposed interface has a handler; a startup self-check prints any that are missing (libwayland-server aborts the compositor on a NULL handler)
- [x] SIGTERM/SIGINT shut down promptly and remove the socket + lockfile (signal via wl_event_loop_add_signal, not sigaction)
- [x] wl_shm.release and wl_surface.get_release (v7) handlers
- [x] wl_keyboard.key sends the raw evdev code (clients add the +8 xkb offset, as term/railfm/railclient do); sending evdev + 8 shifted every key by 8 (Enter arrived as 'j')
- [x] kernel autorepeat (input_event.value == 2) is dropped: clients own repeat via wl_keyboard.repeat_info, so forwarding it doubled every held key
- [x] railclient.c honours wl_keyboard.modifiers (railwl never forwards modifier key events, so ignoring it left shift/ctrl/alt/super permanently off)
- [x] keymap fd includes the trailing NUL (clients mmap and parse it as a C string)
- [x] wl_surface.frame: replace a pending callback instead of clobbering it, and clear s->frame_cb on client destroy (was use-after-free in send_frame_cbs)
- [x] attach(NULL) + commit now unmaps/hides the surface instead of keeping stale content
- [x] xdg_toplevel.resize edges (1/2/4/8) converted to rail drag bits (32/8/16/2); were fed through unmapped
- [x] configure carries real states: TILED_L/R/T/B for tiles, ACTIVATED for the focused one
- [x] real pointer: wl_pointer.set_cursor drawn at pointer-hotspot, NULL hides; built-in arrow is the fallback
- [x] primary selection (zwp_primary_selection_device_manager_v1) — select-to-copy / middle-click paste
- [x] wl_data_offer freed on destroy; clipboard offer pushed to devices that bind late
- [x] pointer axis source + discrete for REL_WHEEL, plus REL_HWHEEL (horizontal scroll)
- [x] real xkb state (xkb_state_update_key) for reported modifiers: AltGr/level-3 and latched/locked now correct, not a hand-built mask
- [x] fullscreen (xdg_toplevel.set_fullscreen): covers the output, bar drawn *under* it, FULLSCREEN state sent; unset returns to the strip
- [x] set_maximized/unset_maximized tracked and reported as MAXIMIZED (tiles already fill the work area)
- [x] wl_output v4: name + description sent (clients that identify outputs by name)

## Tier 3 — desktop plumbing
- [ ] config file (keybinds, bar, gaps)
- [ ] autostart on launch
- [ ] clipboard (primary + clipboard via data-control or wl-clipboard bridge)
- [x] screenshots: ext-image-copy-capture-v1 (wl_shm, output source, cursor optional); client is `shot` (railshot)
- [ ] railnotify as real persistent daemon
- [ ] raillauncher: pass env, spawn into correct seat

## Tier 4 — multi-monitor + session
- [ ] logind/seatd session for DRM master without root
- [ ] VT switching
- [ ] multiple outputs + hotplug
- [ ] per-output workspaces / strip

## Notes
- `setcrtc: Permission denied` when another session holds DRM master —
  run from a free TTY to own the screen.
- Clipboard check (stock source + both readers):
  `printf 'data' | wl-copy` then `wl-paste` and `./build/clip_test`; repeat with
  `--primary` on both. A reader only receives the selection while it holds
  keyboard focus, so `clip_test` maps a throwaway 1x1 toplevel just like
  wl-clipboard does.
- Build: `ninja -C build`
