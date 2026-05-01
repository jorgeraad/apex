# Demo 16 — Customizing Apex: themes, keybindings, your terminal, your style

**Length:** 90 seconds
**Format:** Tight screencast, music-driven
**Audience:** Developers, general audience, lifestyle/personality content
**Series:** Standalone (works as a "subscribe-bait" social clip)

---

## Purpose

Personality demo. Apex ships with 18 themes and a configurable
keybindings layer — that's a vibe-driven feature most security tools
ignore. The viewer should walk away with a small smile and a feeling that
"this thing was built by people with taste."

The clip also re-establishes the *default* theme (`apex` dark) for the
rest of the series so anyone watching demos out of order sees the
canonical look.

---

## Requirements

- **Recording surface:** clean Terminal.app, no notifications, no docker
  status icons, no oh-my-zsh prompt drift.
- **All 18 themes** must render without breaking. Run them through once
  before recording — open a finding, open a dialog, scroll a long output.
- **Keybindings cheat-sheet overlay:** small bottom-right card.
- **Music bed:** royalty-free, loungey, 90 BPM.
- **Asset:** any target with a long-running view (operator session with
  a few queued findings) so cycling themes shows lots of color.
- **Resolution:** 1920×1080 60fps; vertical re-cut for social.

---

## Things to check first

- [ ] Every theme listed in `src/tui/theme/themes/` opens cleanly. Read
      the directory and run them all by hand the morning of recording.
- [ ] Light/dark mode switch (`/themes mode dark|light|auto`) toggles
      cleanly without a flash.
- [ ] No theme has a contrast issue that makes text unreadable on
      camera.
- [ ] Ascii spinner / animation chrome doesn't get clipped at smaller
      terminal sizes if we resize on camera.
- [ ] Default theme set back to `apex` dark by the end. The last frame
      sets the canon.
- [ ] No real target hostnames or PII visible during the cycle. Use
      `vuln-shop` or static demo content.

---

## Full script

### Scene 1 — Cold open (0:00–0:08)

**Visual:** TUI in `apex` dark. The session is already going — a finding
sits on screen.

**VO:** "Apex ships with eighteen themes. Let's look at all of them."

---

### Scene 2 — The cycle (0:08–1:05)

Open the theme picker:

```
/themes
```

Cycle through theme list with arrow keys at one theme per second. Pause
~1.5 seconds on each. The picker shows live previews so the whole TUI
shifts color as we move.

Order suggestion (visually varied):

1. apex
2. ayu
3. catppuccin
4. dracula
5. everforest
6. flexoki
7. github
8. gruvbox
9. kanagawa
10. material
11. monokai
12. nightfox
13. nord
14. onedark
15. rose-pine
16. solarized
17. tokyonight
18. vesper

**VO:** "Catppuccin if you grew up on Discord. Gruvbox if you're a
neovim person. Solarized if you remember 2011. Tokyonight if you have a
mechanical keyboard."

(Keep the line light — these are friendly stereotypes, not insults.)

---

### Scene 3 — Light mode (1:05–1:20)

```
/themes mode light
```

The TUI flips to light. Show a finding rendering cleanly in light mode.

**VO:** "Light mode if you work outside or you're trying to project
onto a wall. `auto` follows the system."

```
/themes mode auto
```

---

### Scene 4 — Keybindings (1:20–1:30)

Cheat-sheet overlay slides in:

```
↑/↓        navigate
Tab        complete
Esc        close dialog
/          slash command
?          help
```

**VO:** "Keybindings work the way you'd expect. Slash for commands. Tab
to complete. Escape to close. Help is always one keystroke away."

---

### Scene 5 — Settle (1:30–1:35)

Cycle back to `apex` dark:

```
/themes apex
/themes mode dark
```

Final frame: `apex` dark, finding visible.

**VO:** "The default. We'll be back in this one for everything else."

End card.

---

## Editing notes

- This is a vibes video. Don't over-narrate. A line per scene is enough.
- Theme cycle is the entire point — make it crisp. Beat-match each cut
  to a soft drum tick in the music.
- Keep total runtime under 95 seconds. This is meant to be re-watchable
  and shareable.
- The vertical re-cut is the social-friendly version: keep Scenes 2 and
  5, drop the rest.
- After this video drops, all subsequent demos use `apex` dark unless
  the demo is *about* themes.
