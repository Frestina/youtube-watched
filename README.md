# youtube-watched

Two small scripts that mark YouTube videos as **watched** on your YouTube
account, so they get a red progress bar and show up in your watch history.
Nothing is downloaded; they log in with your browser's YouTube cookies.

| Command | Script | What it does |
|---|---|---|
| `watched` | `yt-mark-watched` | Marks every video your Kodi box has played as watched, then removes it from Kodi's queue. Runs every hour by itself. |
| `watched-playlist` | `yt-mark-playlist-watched` | Marks every video in one of your YouTube playlists as watched, then removes it from the playlist. You run it by hand. |

Both only remove a video after marking it watched worked. If marking fails, the
video stays where it was and is tried again on the next run.

This guide sets things up on the desktop. For `watched` it assumes your Kodi
box is already set up (see [Before you start](#before-you-start)).

---

## How it works

**`watched` (Kodi queue)**

1. Over SSH it asks Kodi which videos it has played (`ssh <kodi> list`).
2. It opens each video with yt-dlp, logged in with your browser's YouTube
   cookies, and tells YouTube you watched it.
3. When marking works, it removes that video from Kodi's queue
   (`ssh <kodi> rm <name>`). When marking fails, the video stays in the queue
   and is tried again on the next run.

A video that is in the queue several times is marked once, and every copy is
removed.

**`watched-playlist` (YouTube playlist)**

1. It lists the videos in your playlist.
2. It marks each one watched the same way as above.
3. When marking works, it removes the video from the playlist (every copy, if
   it's in there more than once).

YouTube has no official way to remove a playlist video with a browser login,
so step 3 uses the same internal request the YouTube website uses. If YouTube
changes that request, removing may stop working until the script is updated.
Marking watched keeps working either way.

---

## Before you start

You need:

- **A Linux desktop that uses systemd.** This guide is written for
  Arch/Omarchy. On other distros only the package names change.
- **A browser that is logged in to YouTube.** Brave is the default. Chromium,
  Chrome and Firefox also work.
- **For `watched` only: a Kodi box with the queue already set up.** Kodi has to
  write played videos to its queue and allow the SSH commands `list` and `rm`.
  Your desktop's SSH public key must be on the Kodi box (step 4 below).

---

## 1. Install the packages

```bash
sudo pacman -S git yt-dlp python-secretstorage
```

- **yt-dlp** talks to YouTube. Keep it up to date (`sudo pacman -Syu`),
  because YouTube changes often and old versions stop working.
- **python-secretstorage** is easy to forget, but you need it. Brave and other
  Chromium browsers encrypt their cookies with a key kept in
  gnome-keyring. Without this package yt-dlp can't read that key and fails
  with `secretstorage not available`. Firefox doesn't need it.

Check that Python can find yt-dlp:

```bash
/usr/bin/python3 -c 'import yt_dlp; print(yt_dlp.version.__version__)'
```

---

## 2. Get the scripts

This guide puts them in `~/scripts/youtube-watched/`:

```bash
mkdir -p ~/scripts
git clone https://github.com/Frestina/youtube-watched.git ~/scripts/youtube-watched
```

To update later: `git -C ~/scripts/youtube-watched pull`.

---

## 3. Fill in your settings

Your own details (the Kodi address and your playlist) go in a settings file
outside the repository:

```bash
mkdir -p ~/.config/youtube-watched
cp ~/scripts/youtube-watched/settings.example ~/.config/youtube-watched/settings
```

Open `~/.config/youtube-watched/settings` and fill in:

```
host = root@<kodi-ip>
playlist = https://www.youtube.com/playlist?list=<playlist-id>
```

- `host` is the SSH user and IP address of your Kodi box, for example
  `root@192.168.1.20`. Only `watched` uses it.
- `playlist` is the playlist `watched-playlist` empties. Open the playlist on
  YouTube and copy the address from the browser. Only `watched-playlist` uses
  it.

You can also pass either one on the command line instead (see
[Usage](#7-add-the-commands)), but the hourly timer uses the settings file.

### Using a browser other than Brave?

The default browser is `brave`. To use another one, pass it as the second
argument (`watched root@<kodi-ip> chromium`), or change the default in the
script:

```python
browser = args[1] if len(args) > 1 else 'brave'
```

---

## 4. Set up SSH to Kodi (only for `watched`)

The script runs SSH with `BatchMode=yes`, so SSH never stops to ask for
anything. That keeps the hourly job from hanging, but it also means every
prompt must already be dealt with.

### 4a. Have an SSH key

```bash
ls ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519
```

Press Enter at the passphrase question to leave it empty. The timer runs with
nobody there to type a passphrase.

Send the contents of `~/.ssh/id_ed25519.pub` to whoever set up the Kodi box,
so the key can be added there.

### 4b. Add a Host block to `~/.ssh/config`

This tells SSH exactly which key to use for Kodi and keeps any SSH agent out of
it. **Don't skip this if you use a password-manager SSH agent** (Bitwarden,
1Password, …). Without it, SSH offers the agent's keys and you get an
approval popup every hour, or the job fails while the vault is locked.

Back up the file first, then add the block at the end (replace `<kodi-ip>`
with your Kodi box's IP address, in both places):

```bash
cp -p ~/.ssh/config ~/.ssh/config.bak.$(date +%Y%m%d) 2>/dev/null
cat >> ~/.ssh/config <<'EOF'

Host kodi <kodi-ip>
  HostName <kodi-ip>
  User root
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  IdentityAgent none
EOF
chmod 600 ~/.ssh/config
```

The `Host` line must include the **IP address**, because the script connects
to `root@<kodi-ip>`, not to `kodi`.

### 4c. Accept Kodi's host key (once)

The first connection asks you to trust the Kodi box. Do it by hand once:

```bash
ssh root@<kodi-ip> list
```

Type `yes`. You should then see one line per queued video (or nothing if the
queue is empty). If you get `Permission denied`, Kodi doesn't have your key
yet (step 4a).

---

## 5. Make sure your browser is logged in to YouTube

Open YouTube in your browser and check that you're logged in to the
account whose history you want updated. Then check that yt-dlp sees the
login (change `brave` to your browser):

```bash
/usr/bin/python3 -c "
from yt_dlp import YoutubeDL
ydl = YoutubeDL({'cookiesfrombrowser': ('brave', None, 'GNOMEKEYRING', None), 'quiet': True})
print('logged in:', ydl.get_info_extractor('Youtube').is_authenticated)"
```

It should print `logged in: True`. For Firefox, replace `'GNOMEKEYRING'` with
`None`.

Why `GNOMEKEYRING` is named: under Hyprland (Omarchy) yt-dlp can't work out
which keyring the browser uses, so the scripts name it. If your desktop is
KDE, change `'GNOMEKEYRING'` in the scripts to `'KWALLET6'` (or `'KWALLET5'`).

---

## 6. Test it

**Dry run first.** This marks the videos watched but removes nothing:

```bash
~/scripts/youtube-watched/yt-mark-watched --dry-run
~/scripts/youtube-watched/yt-mark-playlist-watched --dry-run
```

Example output:

```
✓ 1/2  Some Video Title  (watched, dry run: would remove 20261023-144201-nBXpFbOPUdA)
✓ 2/2  Another Video  (watched, dry run: would remove 20261023-144647-71ljFVfNN8U)
```

Now open your [YouTube watch history](https://www.youtube.com/feed/history).
The videos should be there with red progress bars. If they are, do a real
run (without `--dry-run`).

---

## 7. Add the commands

So you can type `watched` and `watched-playlist` from any folder:

```bash
mkdir -p ~/.local/bin
for name in watched watched-playlist; do
  type $name 2>/dev/null && echo "WARNING: '$name' already exists, pick another name"
done
ln -s ~/scripts/youtube-watched/yt-mark-watched ~/.local/bin/watched
ln -s ~/scripts/youtube-watched/yt-mark-playlist-watched ~/.local/bin/watched-playlist
```

`~/.local/bin` is in your PATH on Omarchy by default. If `watched` gives
"command not found", add this to `~/.bashrc` and open a new terminal:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Usage:

```bash
watched                              # mark watched and remove from Kodi's queue
watched --dry-run                    # mark watched only, remove nothing
watched root@<kodi-ip> [browser]     # use another Kodi box / browser this time

watched-playlist                     # mark watched and remove from the playlist
watched-playlist --dry-run           # mark watched only, remove nothing
watched-playlist <playlist> [browser]   # use another playlist (URL or ID) / browser
```

The links point at the scripts, so a `git pull` updates the commands too.

---

## 8. Run `watched` every hour (systemd timer)

This has to be a **user** timer (`systemctl --user`), not a system one. A user
timer runs inside your logged-in session, where it can get into
gnome-keyring and your browser's cookies. One downside: it only runs while
you're logged in, but that's when the cookies can be read anyway.

Create the service. `%h` means your home folder, so fix the path if you put
the scripts somewhere other than `~/scripts/youtube-watched/`:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/yt-mark-watched.service <<'EOF'
[Unit]
Description=yt-mark-watched: mark videos Kodi played as watched on YouTube

[Service]
Type=oneshot
ExecStart=/usr/bin/python3 %h/scripts/youtube-watched/yt-mark-watched
Nice=10
EOF
```

Create the timer:

```bash
cat > ~/.config/systemd/user/yt-mark-watched.timer <<'EOF'
[Unit]
Description=yt-mark-watched: hourly

[Timer]
OnCalendar=hourly
RandomizedDelaySec=5min
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

- `RandomizedDelaySec=5min` makes each run start up to 5 minutes after the
  hour, at a random time.
- `Persistent=true` means a run that was missed while the PC was off happens
  soon after it starts.

Turn it on and test it once:

```bash
systemctl --user daemon-reload
systemctl --user enable --now yt-mark-watched.timer
systemctl --user start yt-mark-watched          # run the job right now
journalctl --user -u yt-mark-watched -n 20      # see what it did
systemctl --user list-timers yt-mark-watched.timer   # when it runs next
```

`watched-playlist` has no timer; run it whenever you want the playlist
emptied.

---

## Reading the output

| Line | Meaning |
|---|---|
| `✓ … (watched, removed from queue)` / `(…from playlist)` | Done. |
| `✗ … (marking watched failed, kept in …)` | YouTube refused. It's tried again next run. |
| `✗ … (could not open video, kept in …)` | Video deleted, private, or yt-dlp is outdated. |
| `~ … (watched, but removing from … failed: …)` | Marked, but not removed. The next run marks it again (harmless) and tries removing it again. |
| `✗ could not parse queue entry, kept: …` | Kodi's queue has a line the script doesn't understand. |
| `Queue is empty, nothing to do.` / `Playlist "…" is empty, nothing to do.` | Nothing to mark. |
| `Kodi not reachable, skipping` | The Kodi box is off. This is normal, and the run still counts as a success. |

Exit code: **0** if everything worked (or Kodi was off), **1** if anything
failed. In the timer's log, a failed run shows as `Failed with result
'exit-code'`.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `No Kodi host: …` / `No playlist: …` | Fill in `~/.config/youtube-watched/settings` (step 3). |
| `secretstorage not available` | `sudo pacman -S python-secretstorage` (step 1). |
| `Not logged in to YouTube in brave, nothing marked.` | Log in to YouTube in that browser. If you use more than one browser profile, the scripts read the **default** profile. |
| `Could not load brave cookies: …` | Wrong browser name, or the keyring is locked. Log in to the desktop normally so gnome-keyring unlocks. |
| `Could not read the playlist …` | Wrong playlist address, or the playlist belongs to another account than the one logged in. |
| `Could not read the Kodi queue: Host key verification failed.` | Do step 4c once by hand. If Kodi was reinstalled, its host key changed: `ssh-keygen -R <kodi-ip>`, then step 4c again. |
| `Could not read the Kodi queue: Permission denied …` | Kodi doesn't have your public key (step 4a), or the Host block points at the wrong key. |
| Password-manager popup every hour | The Host block (step 4b) is missing, or its `Host` line doesn't include the IP. Check with `ssh -G root@<kodi-ip> \| grep -i identity`; it should show `identityagent none`. |
| Works by hand but not from the timer | Check `journalctl --user -u yt-mark-watched`. Usually the keyring wasn't unlocked yet, or the path in `ExecStart` is wrong. |
| Lots of `✗ could not open video` | Update yt-dlp: `sudo pacman -Syu`. |

---

## Uninstall

```bash
systemctl --user disable --now yt-mark-watched.timer
rm ~/.config/systemd/user/yt-mark-watched.{service,timer}
systemctl --user daemon-reload
rm ~/.local/bin/watched ~/.local/bin/watched-playlist
rm -r ~/scripts/youtube-watched ~/.config/youtube-watched
```

Then remove the `Host kodi …` block from `~/.ssh/config` if you no longer need
it.
