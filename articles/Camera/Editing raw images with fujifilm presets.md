**For RawTherapee**

- **[GLTR87/RawTherapee-presets-Fuji-inspired](https://github.com/GLTR87/RawTherapee-presets-Fuji-inspired)** (GitHub, free, public domain) — presets directly inspired by Fuji's film simulation modes, covering tone curves, HSV adjustments, and B&W/desaturation for the Acros/Monochrome-style looks. Calibrated on a Sony A7 but the maker notes it works well on Canon and Olympus RAW files too, so it should behave reasonably on your Pixel DNGs or an LX5's RAW output.
- **Stuart Sowerby's Film Simulation LUTs/profiles** — the most-cited original source that most of the other Fuji-style packs (including the Darktable ones below) are actually built from or ported from. Worth searching for directly if you want the "root" version.

**For Darktable**

- **[t3mujinpack](https://t3mujinpack.github.io/)** (GitHub) — probably the most comprehensive free pack out there. Covers actual named film stocks: Fuji Velvia, Fuji Pro, Fuji Astia/Provia, plus Kodak Portra/Ektachrome/Ektar/Kodachrome, Agfa Vista, and Ilford B&W. This is broader than just Fuji simulations — real analog film emulation across brands.
- **Darktable.fr's Fujifilm X styles pack** — a free zip specifically porting the 15 official Fuji X-Trans III film simulations (Provia, Velvia, Astia, Classic Chrome, Pro Neg Hi/Std, Acros variants, Monochrome, Sepia) into Darktable style format, built by Jean-Paul Gauche and Andy Costanza from Sowerby's original RawTherapee work — so this is genuinely the closest thing to "actual Fuji simulations" you can get on non-Fuji gear.
- **[bastibe's Darktable Film Simulation Panel](https://github.com/bastibe/Darktable-Film-Simulation-Panel)** — a more advanced/technical option that lets you build your own custom simulation styles using color-checker calibration, including mixing brands (their example: applying Ricoh's "Positive Film" look to a Fuji photo).

**How to actually use them (same basic idea for both programs):**

1. Download the preset/style pack (usually a `.zip` with `.pp3` files for RawTherapee, or a `.dtstyle`/style import for Darktable).
2. In RawTherapee: reset to the "Neutral" profile first, then load the preset via the Profiles panel.
3. In Darktable: import via the **Styles module** in the lighttable view, then apply to your image in darkroom mode.
4. Both let you tweak further after applying — presets are a starting point, not a locked final result.

**One practical note given your Pixel 9 Pro RAW files specifically:** since phone DNGs go through some computational processing before you even get the RAW, results may not look identical to how these same presets behave on a "cleaner" single-exposure RAW from something like the LX5 — worth testing on a few different photos to see how much the underlying computational photography fights against the film-grain/imperfection look versus a genuine CCD file.