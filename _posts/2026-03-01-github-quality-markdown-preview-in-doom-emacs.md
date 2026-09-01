---
title: GitHub-Quality Markdown Preview in Doom Emacs
date: 2026-03-01 12:00:00 -0500
categories: [utilities]
tags: [emacs, doom-emacs, markdown, macos]
img_path: /assets/img/
---

One thing I kept falling back to VS Code for was markdown preview. `Cmd+Shift+V` in VS Code gives you a clean, live-updating, side-by-side rendered preview with syntax-highlighted code blocks and proper formatting. I wanted the same experience in Doom Emacs without leaving my editor.

It took some digging, but I got it working — full GitHub-flavored rendering with syntax highlighting, inline code styling, and a split-pane layout. Here's exactly how.

### What we're building

- **GitHub-quality rendering** via [grip](https://github.com/joeyespo/grip), which uses GitHub's own Markdown API
- **Embedded preview inside Emacs** using xwidget-webkit (an embedded Chromium-based browser)
- **Side-by-side split** — source on the left, rendered preview on the right
- **Toggle with `Cmd+Shift+V`** — same muscle memory as VS Code

### Prerequisites

- **Emacs 30+** compiled with xwidget support (via [emacs-plus](https://github.com/d12frosted/homebrew-emacs-plus))
- **Doom Emacs**
- **grip** — Python-based GitHub Markdown renderer
- **GitHub CLI** (`gh`) — for API authentication
- **pipx** — for installing Python CLI tools cleanly

### Step 1: Install Emacs with xwidget support

If you're using `emacs-plus` on macOS, you need the `--with-xwidgets` flag. Without it, Emacs can't embed a webkit browser for the preview pane.

In your `Brewfile`:

```ruby
brew 'd12frosted/emacs-plus/emacs-plus@30', args: ["with-xwidgets"]
```

If Emacs is already installed without xwidgets, reinstall:

```bash
brew reinstall d12frosted/emacs-plus/emacs-plus@30 --with-xwidgets
```

You can verify xwidget support is enabled:

```bash
/opt/homebrew/opt/emacs-plus@30/Emacs.app/Contents/MacOS/Emacs --batch \
  --eval '(princ (featurep (quote xwidget-internal)))'
```

This should print `t`. If it prints `nil`, the rebuild didn't include xwidgets.

### Step 2: Install grip

grip is the engine that renders your markdown. It sends your file to GitHub's Markdown API and serves the rendered HTML locally with GitHub's actual CSS.

```bash
brew install pipx
pipx install grip
```

### Step 3: Patch grip's template (critical)

This is the part that took the most debugging. grip 4.6.2 fetches GitHub's latest CSS files, but its HTML template is missing the `data-color-mode` attributes that GitHub's modern CSS requires to activate. Without this patch, all the CSS loads but no styles apply — you get flat, unstyled HTML.

Find grip's base template:

```bash
find ~/.local/pipx/venvs/grip -name "base.html" -path "*/templates/*"
```

Edit that file and change:

```html
<html lang="en">
```

to:

```html
<html lang="en" data-color-mode="auto" data-light-theme="light" data-dark-theme="dark">
```

Or do it in one shot:

```bash
GRIP_BASE_TMPL=$(find ~/.local/pipx/venvs/grip -name "base.html" -path "*/templates/*")
sed -i '' 's/<html lang="en">/<html lang="en" data-color-mode="auto" data-light-theme="light" data-dark-theme="dark">/' "$GRIP_BASE_TMPL"
```

> **Note:** This patch gets overwritten on `pipx upgrade grip`. I automate it in my [dotfiles install.sh](https://github.com/jbmartino/dotfiles) so it's reapplied automatically.

### Step 4: Set up GitHub authentication

grip uses the GitHub Markdown API, which is rate-limited to 60 requests/hour without authentication. With a token, you get 5,000 requests/hour.

The easiest approach if you already use `gh` CLI:

```bash
# In .zshrc
export GITHUB_TOKEN="$(gh auth token 2>/dev/null)"
```

Then configure grip to use it in your Doom config (since GUI Emacs doesn't inherit shell environment variables):

```elisp
;; In config.el
(after! grip-mode
  (setq grip-github-user "your-github-username"
        grip-github-password (string-trim (shell-command-to-string "gh auth token"))))
```

### Step 5: Configure Doom Emacs

**`init.el`** — Enable the `+grip` flag on the markdown module:

```elisp
(markdown +grip)  ; writing docs for people to ignore
```

**`config.el`** — Set up xwidget preview with the VS Code keybinding:

```elisp
;; Markdown preview with grip-mode (GitHub-flavored rendering via GitHub API)
(after! grip-mode
  (setq grip-github-user "your-github-username"
        grip-github-password (string-trim (shell-command-to-string "gh auth token")))
  (setq grip-preview-use-webkit t)
  (add-to-list 'display-buffer-alist
               '("\\*xwidget" (display-buffer-in-side-window)
                 (side . right) (slot . 0) (window-width . 0.5))))

(defun my/markdown-preview-toggle ()
  "Toggle grip markdown preview in a right split."
  (interactive)
  (if (and (boundp 'grip-mode) grip-mode)
      (progn
        (grip-mode -1)
        (when-let ((buf (seq-find
                         (lambda (b) (with-current-buffer b
                                       (eq major-mode 'xwidget-webkit-mode)))
                         (buffer-list))))
          (delete-windows-on buf)
          (kill-buffer buf)))
    (delete-other-windows)
    (require 'grip-mode)
    (grip-mode 1)))

(map! :after markdown-mode
      :map markdown-mode-map
      "s-V" #'my/markdown-preview-toggle)
(map! :after org
      :map org-mode-map
      "s-V" #'my/markdown-preview-toggle)
```

Run `doom sync` after updating init.el:

```bash
~/.config/emacs/bin/doom sync
```

### Step 6: Use it

1. Open any markdown file in Doom Emacs
2. Hit `Cmd+Shift+V`
3. A split opens with the rendered preview on the right
4. Hit `Cmd+Shift+V` again to close the preview

Doom also maps grip to `SPC m p` (local leader → preview) in markdown buffers.

### Why not just use markdown-mode's built-in preview?

Doom's markdown-mode includes `markdown-live-preview-mode` which renders in an eww buffer. The rendering quality is basic — no syntax highlighting in code blocks, no GitHub styling, and the CSS is minimal. If you're writing docs that will end up on GitHub, you want to see exactly what GitHub will render.

### Why not markdown-preview-mode?

`markdown-preview-mode` renders locally using marked.js in a browser tab. It's decent and works offline, but it opens in an external browser rather than inside Emacs. You lose the integrated split-pane experience.

### The grip template bug

The most frustrating part of this setup was grip's rendering looking completely flat despite having ~25 GitHub CSS files loaded. The root cause: GitHub's modern CSS uses `data-color-mode` selectors on the `<html>` element to activate themes. grip's HTML template hasn't been updated to include these attributes. The CSS loads, the HTML renders, but no styles match — so everything looks like unstyled HTML.

Adding `data-color-mode="auto" data-light-theme="light" data-dark-theme="dark"` to the `<html>` tag is a one-line fix that makes everything work. I've opened this as an issue upstream, but in the meantime the `sed` patch in step 3 handles it.

### Full dotfiles

All of this is automated in my [dotfiles](https://github.com/jbmartino/dotfiles) — the Brewfile, install.sh, and Doom config handle everything including the grip template patch. A fresh `install.sh` run sets up the complete markdown preview pipeline.
