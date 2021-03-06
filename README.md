# bin
This repository contains a collection of scripts and related files that I am using on my Arch Linux installation. There
might be better ways to do some of the things I wrote these scripts for, or even already existing solutions I simply
didn't know about; if you think you know of either one please let me know.


## restart-plasmashell
Sometimes the KDE Plasma shell starts displaying my wallpaper strangely (i.e. not at all) towards the bottom of my
screen. When that happens I simply run this script to quickly restart the shell.


## vpn-whitelist-domain
I am using NetworkManager and the NetworkManager-OpenVPN plugin. When connecting to my VPN provider, NetworkManager
creates a virtual tunnel-device through which all traffic is routed by setting the route metric lower than the metric
of my default gateway.

Some sites and services unfortunately have simply blocked or banned my VPN's IP address, so for some domains it is
useful to bypass the VPN tunnel entirely. That is what this script does: it finds the first default gateway that isn't a
tunnel device, resolves the passed domain to all possible IPs and then adds explicit routes for each of them through the
discovered gateway.

```
Usage:
    vpn-whitelist-domain [SWITCHES] domain

Meta-switches
    -h, --help         Prints this help message and quits
    --help-all         Print help messages of all subcommands and quit
    -v, --version      Prints the program's version and quits

Switches
    -r, --remove       Remove domain from whitelist
```


## warm-up-dns-resolver
I am using Unbound with DNSSEC enabled. The resolving of an uncached domain takes quite a bit of time because of that,
and most domains have a low TTL or even a TTL of 0 set which disables caching in Unbound completely. Naturally this is
a bit annoying, so I wrote this script.

```
usage: warm-up-dns-resolver [-h] [--file PATH [PATH ...]] [--use-firefox]
                            [--firefox-profile PATH] [--days DAYS]
                            [domains [domains ...]]

positional arguments:
  domains

optional arguments:
  -h, --help            show this help message and exit
  --file PATH [PATH ...]
                        Paths to files containing domains.
  --use-firefox         Use domains in Mozilla Firefox's history.
  --firefox-profile PATH
                        Specify the path to a profile folder.
  --days DAYS           Use domains of history up to n days before the last
                        made entry.
```

It can also scrape domains directly from Mozilla Firefox's history. The matching systemd *.service and *.timer files
execute this command: `warm-up-dns-resolver --file ~/.warm-up-dns-resolver-domains --use-firefox`. This allows me to
place high-priority domains into a file in my home directory and also always quickly resolve domains that I visited in
the last 7 days using Firefox. Additionally I adapted my `unbound.conf` a bit:

```
server:
  use-syslog: yes
  username: "unbound"
  directory: "/etc/unbound"
  trust-anchor-file: trusted-key.key
  root-hints: "/etc/unbound/root.hints"

  cache-min-ttl: 3600
  prefetch: yes
  prefetch-key: yes
```

`cache-min-ttl: 3600` forces caching of resolved domains to at least one hour. Due to enabling `prefetch`, Unbound will
start resolving cached domains as soon as their remaining TTL falls below 10% -- that's why I configured the systemd
timer unit to run every 6 minutes. In theory this should keep every specified domain cached in Unbound.


## systemd-octor
```
usage: systemd-octor [-h] [--output OUTPUT] [--compare COMPARE COMPARE]

optional arguments:
  -h, --help            show this help message and exit
  --output OUTPUT, -o OUTPUT
                        Change destination of output file
  --compare COMPARE COMPARE, -c COMPARE COMPARE
                        Compare first output to second output
```

This is used to compare the systemd service configuration between two systems. Useful for example in case some important
units were disabled on system A and the exact changes made were forgotten. An easy way to fix the configuration then is
to create a fresh Arch Linux installation (system B) in a `systemd-nspawn` container, or as a full virtual machine, and
running this script to get a sane systemd service configuration which can then be compared using:
`systemd-octor --compare A.json B.json`.


## fix-openvpn
`# systemctl enable fix-openvpn@<syslog-identifier>`
```
usage: fix-openvpn [-h] syslog-identifier [gateway]

positional arguments:
  syslog-identifier  The syslog identifier of the journal to monitor
  gateway            Specify the gateway. If not set, the script will attempt
                     to determine the default gateway

optional arguments:
  -h, --help         show this help message and exit
```

Monitors the OpenVPN systemd log and automatically adds routes through the default gateway for remote link IPs that are
attempted to be connected to. This is needed when the resolved IP address for a specified VPN domain changes, preventing
automatic reconnects to the VPN server, because the traffic is routed through a broken connection.

Note: It seems that OpenVPN already attempts to do that on its own, but somehow fails. It might be using the old IP
address after re-establishing the tunnel.


## [aur-auto-vote](https://www.reddit.com/r/archlinux/comments/4ryh6t/aur_autovote/)
I really wanted to show my appreciation for the AUR packages that I am using, but am entirely too lazy to keep the list
of voted-for packages up-to-date manually.

```
$ aur-auto-vote --help
usage: aur-auto-vote [-h] [--ignore IGNORE] [--unvote-all] [--delay DELAY]
                     username

positional arguments:
  username

optional arguments:
  -h, --help            show this help message and exit
  --ignore IGNORE, -i IGNORE
                        Regex for packages that should not be voted. Can be
                        passed multiple times.
  --unvote-all, -u      Unvote all voted-for packages, all other arguments are
                        ignored.
  --delay DELAY, -d DELAY
                        Delay between voting actions (seconds).
```

When voting already voted-for packages will be automatically skipped. Local PKGBUILDs I usually prefix with my nick,
i.e. `cryzed-` so I pass in `--ignore 'cryzed-.*'`. If you feel that your voted-for packages are completely outdated,
consider using `--unvote-all` to remove all votes beforehand. It might be a good idea to specify a `--delay` > 0 to
prevent hammering the server.


## backup-system
* `# systemctl start backup-system` (to start the script via systemd. A systemd `EnvironmentFile` with a `DIRECTORY` key
is needed at `/etc/backup-system.conf`)
* `# backup-system [DESTINATION]` (to start the script manually, destination is optional)
* `# systemctl enable backup-system.timer` (to enable the timer for daily backups)

This is a script that backups the most important parts of my Arch Linux installation, should it ever break in an
unrecoverable fashion. It uses:

* [`lostfiles`](https://aur.archlinux.org/packages/lostfiles/) to get a list of files in the system which are not part
of any package, i.e. files I most likely created and added to the system without using a custom PKGBUILD.
* `pacman -Qii | awk '/^MODIFIED/ {print $2}'` to get a list of modified backup files, i.e. modified configuration files
of installed packages
* `pacman -Q --explicit` to save the list of all explicitly installed packages
* all systemd overrides located in `/etc/systemd/system` or `/etc/systemd/user`
* `systemd-octor` (see above) to save the systemd service configuration
* all `/home` directories
* `/root`

and uses `tar` to compress all those files into an archive with the current date.

Note that I have symlinked all XDG directories, i.e. "Documents", "Downloads", "Music" etc. to a different HDD to
prevent unnecessary writes on my system SSD -- so when backing up the `/home` directories, tar will only backup the
dotfiles and not automatically follow the symlinks. The resulting backup file is usually around ~10-12 GB in size.


## defaults
```
$ ./defaults --help
usage: defaults [-h] [--editor PATH] [--stdout] path

Inspect the original version of files belonging to installed packages.

If the editor is not specified using --editor, the environment variables EDIT
and VISUAL will be used; alternatively --stdout can be used to pipe the original
file's contents directly to stdout.

Files are searched for in packages located in the "CacheDir" directories specified
in /etc/pacman.conf (default: /var/cache/pacman/pkg/) and in the "PKGDEST"
directory specified in /etc/makepkg.conf, if defined. Defaults will only work
with foreign packages, if the "PKGDEST" directory is specified.

If the package can't be found in either, defaults attempt to download the latest
version of the package from the available pacman mirrors, however this only
works with native packages.

positional arguments:
  path

optional arguments:
  -h, --help     show this help message and exit
  --editor PATH
  --stdout
```

This is useful for me when editing backup (configuration)-files which are part of installed packages. Often they are
filled with comments and various examples configurations, however after figuring out my configuration I rarely want to
keep all these around.

Removing them to keep the file as clean as possible however, might be an issue when further modifications are needed at
some later point in time -- all the reference examples are now missing, forcing you to either read the man page (even
for trivial changes) or manually hunt down the package in your cache or online and checking out the original file.

This script aims to save you these steps: simply run it with a path to a file, and the local, as well as the original
file, should both be opened in the specified editor.


## hotstrings
```
$ hotstrings --help
usage: hotstrings [-h] [path]

positional arguments:
  path        Path to JSON file containing hotstring definitions

optional arguments:
  -h, --help  show this help message and exit
```

This script is a much lighter version of AutoKey, specifically the
[AutoKey-py3](https://aur.archlinux.org/packages/autokey-py3/) fork. Since there was a recent update to the official
[python-xlib](https://github.com/python-xlib/python-xlib) which breaks compatibility with the fork, and the fork also
doesn't seem to be actively maintained anymore, I wanted to preemptively find another solution before AutoKey-py3
eventually stops working entirely on my system.

This pretty much implements only the functionality I actually use: replacing hotstrings and running commands; basically
you type text and it is either replaced by the given replacement string or a specified command is run (optionally
replacing the hotstring with its stdout output). Dependencies are Python 3 and the the official python-xlib or
[LiuLang's fork](https://github.com/LiuLang/python3-xlib) which is used by AutoKey-py3.

An example configuration file might look like this:
```
$ cat ~/.config/hotstrings.json 
{
    "first": ["replace", "replacement 1"],
    "second": ["replace", "replacement 2"],
    "third": ["run", "sh", "-c", "touch ~/Desktop/hello_world.txt"],
    "fourth": ["run-replace", "date"],

    // doesn't strip whitespace at the beginning and end of the output
    "five": ["run-replace-raw", "date"]
}
```


## provides
```
$ provides --help
usage: provides [-h] package

positional arguments:
  package

optional arguments:
  -h, --help  show this help message and exit
```

Small utility to list files installed by a package contained in `PATH`.


## youtuber
```
$ youtuber --help
usage: youtuber [-h] [--dont-recurse] path

positional arguments:
  path

optional arguments:
  -h, --help      show this help message and exit
  --dont-recurse
```

Small utility that helps keeping a YouTube video collection up-to-date. The provided path is searched recursively (by
default) for files ending in ".youtube-dl". These youtube-dl files can contain all arguments that the `youtube-dl`
executable would accept (separated by spaces, newlines, or both). `youtuber` then downloads all missing files (with a
nice tqdm-interface) into the directory the youtube-dl-file was found in (unless manually overriden with an absolute
path with the -o/--output option). This allows specifying different options for all youtube-dl-files.

The `youtube-dl` system- and user-configurations are respected and properly merged with all options specified in the
youtube-dl-files.


## kde-bitday
KDE Plasma-compatible wallpaper changer script for the [Bitday wallpaper set](http://danny.care/bitday/download/). Note:
Do not put any files into the target directory, they will be deleted!

```
$ bitday --help
usage: bitday [-h] [--source SOURCE] target

positional arguments:
  target           Path to the KDE Slideshow wallpaper folder

optional arguments:
  -h, --help       show this help message and exit
  --source SOURCE  Path to a folder containing a BitDay 2 wallpaper set
```
