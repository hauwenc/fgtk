fgtk: a set of a misc tools to work with files and processes
--------------------

Various oldish helper binaries I wrote to help myself with day-to-day tasks.



### Files

#### scim set

A set of tools to bind a bunch of scattered files to a single path, with
completely unrelated internal path structure. Intended usage is to link
configuration files to scm-controlled path (repository).

Actually started as [cfgit project](http://fraggod.net/code/git/configit/), but
then evolved away from git vcs into a more generic, not necessarily vcs-related,
solution.

##### scim-ln

Adds a new link (symlink or catref) to a manifest (links-list), also moving file
to scim-tree (repository) on fs-level.

##### scim

Main tool to check binding and metadata of files under scim-tree. Basic
operation boils down to two (optional) steps:

* Check files' metadata (uid, gid, mode, acl, posix capabilities) against
  metadata-list (.scim_meta, by default), if any, updating the metadata/list if
  requested, except for exclusion-patterns (.scim_meta_exclude).

* Check tree against links-list (.scim_links), warning about any files / paths
  in the same root, which aren't on the list, yet not in exclusion patterns
  (.scim_links_exclude).

##### pyacl

Tool to restore POSIX ACLs on paths, broken by chmod or similar stuff without
actually changing them.

##### rsync-diff

Tool to sync paths, based on berkley db and rsync.

Keeps b-tree of paths (files and dirs) and corresponding mtimes in berkdb,
comparing state when ran and building a simple merge-filter for rsync ("+ /path"
line for each changed file/dir, including their path components, ending with "- *").
Then it runs a single rsync with this filter to efficiently sync the paths.

Note that the only difference from "rsync -a src dst" here is that "dst" tree
doesn't have to exist on fs, otherwise scanning "dst" should be pretty much the
same (and probably more efficient, depending on fs implementation) b-tree
traversal as with berkdb.

Wrote it before realizing that it's quite pointless for my mirroring use-case -
I do have full source and destination trees, so rsync can be used to compare (if
diff file-list is needed) or sync them.

##### fs

Complex tool for high-level fs operations. Reference is built-in.

Copy files, setting mode and ownership for the destination:

	fs -m600 -o root:wheel cp * /somepath

Temporarily (1hr) change attributes (i.e. to edit file from user's editor):

	fs -t3600 -m600 -o someuser expose /path/to/file

Copy ownership/mode from one file to another:

	fs cps /file1 /file2

##### fatrace-pipe

[fatrace](https://launchpad.net/fatrace)-based script to read filesystem write
events via linux [fanotify](http://lwn.net/Articles/339253/) system and match
them against specific path and app name, sending matches to a FIFO pipe.

Use-case is to, for example, setup watcher for development project dir changes,
sending instant "refresh" signals to something that renders the project or shows
changes' results otherwise.

FIFO is there because fanotify requires root privileges, and running some
potentially-rm-rf-/ ops as uid=0 is a damn bad idea.
User's pid can read lines from the fifo and react to these safely instead.

Example - run "make" on any change to `~user/hatch/project` files:

	(root) ~# fatrace-pipe ~user/hatch/project
	(user) project% xargs -in1 </tmp/fatrace.fifo make

##### clean-boot

Script to remove older kernel versions (as installed by /sbin/installkernel)
from /boot or similar dir.

Always keeps version linked as "vmlinuz", and prioritizes removal of older
patchset versions from each major one, and only then latest per-major patchset,
until free space goal (specified percentage, 20% by default) is met.

Also keeps specified number of last-to-remove versions, can prioritize cleanup
of ".old" verssion variants, keep `config-*` files... and other stuff (see --help).

Example:

	# clean-boot --debug --dry-run -f 100
	DEBUG:root:Preserved versions (linked version, its ".old" variant, --keep-min): 4
	DEBUG:root: - 3.9.9.1 - System.map-3.9.9-fg.mf_master
	DEBUG:root: - 3.9.9.1 - config-3.9.9-fg.mf_master
	DEBUG:root: - 3.9.9.1 - vmlinuz-3.9.9-fg.mf_master
	DEBUG:root: - 3.10.27.1 - vmlinuz-3.10.27-fg.mf_master
	...
	DEBUG:root: - 3.12.19.1 - System.map-3.12.19-fg.mf_master
	DEBUG:root: - 3.12.20.1 - config-3.12.20-fg.mf_master
	DEBUG:root: - 3.12.20.1 - System.map-3.12.20-fg.mf_master
	DEBUG:root: - 3.12.20.1 - vmlinuz-3.12.20-fg.mf_master
	DEBUG:root:Removing files for version (df: 58.9%): 3.2.0.1
	DEBUG:root: - System.map-3.2.0-fg.mf_master
	DEBUG:root: - config-3.2.0-fg.mf_master
	DEBUG:root: - vmlinuz-3.2.0-fg.mf_master
	DEBUG:root:Removing files for version (df: 58.9%): 3.2.1.0
	... (removal of older patchsets for each major version, 3.2 - 3.12)
	DEBUG:root:Removing files for version (df: 58.9%): 3.12.18.1
	... (this was the last non-latest patchset-per-major)
	DEBUG:root:Removing files for version (df: 58.9%): 3.2.16.1
	... (removing latest patchset for each major version, starting from oldest - 3.2 here)
	DEBUG:root:Removing files for version (df: 58.9%): 3.7.9.1
	...
	DEBUG:root:Removing files for version (df: 58.9%): 3.8.11.1
	...
	DEBUG:root:Finished (df: 58.9%, versions left: 4, versions removed: 66).

("df" doesn't rise here because of --dry-run, "-f 100" = "remove all
non-preserved" - as df can't really get to 100%)

Note how 3.2.0.1 (non-.old 3.2.0) gets removed first, then 3.2.1, 3.2.2, and so
on, but 3.2.16 (latest of 3.2.X) gets removed towards the very end, among other
"latest patchset for major" versions, except those that are preserved
unconditionally (listed at the top).

##### findx

Wrapper around GNU find to accept paths at the end of argv if none are passed
before query.

Makes it somewhat more consistent with most other commands that accept options
and a lists of paths (almost always after opts), but still warns when/if
reordering takes place.

No matter how many years I'm using that tool, still can't get used to typing
paths before query there, so decided to patch around that frustrating issue one
day.

##### wgets

Simple script to grab a file using wget and then validate checksum of the
result, e.g.:

```console
$ wgets -c http://os.archlinuxarm.org/os/ArchLinuxARM-sun4i-latest.tar.gz cea5d785df19151806aa5ac3a917e41c
Using hash: md5
Using output filename: ArchLinuxARM-sun4i-latest.tar.gz
--2014-09-27 00:04:45--  http://os.archlinuxarm.org/os/ArchLinuxARM-sun4i-latest.tar.gz
Resolving os.archlinuxarm.org (os.archlinuxarm.org)... 142.4.223.96, 67.23.118.182, 54.203.244.41, ...
Connecting to os.archlinuxarm.org (os.archlinuxarm.org)|142.4.223.96|:80... connected.
HTTP request sent, awaiting response... 416 Requested Range Not Satisfiable

    The file is already fully retrieved; nothing to do.

Checksum matched
```

Basic invocation syntax is `wgets [ wget_opts ] url checksum`, checksum is
hex-decoded and hash func is auto-detected from its length (md5, sha-1, all
sha-2's are supported).

Idea is that - upon encountering an http link with either checksum on the page
or in the file nearby - you can easily run the thing providing both link and
checksum to fetch the file.

If checksum is available in e.g. *.sha1 file alongside the original one, it
might be a good idea to fetch that checksum from any remote host (e.g. via
"curl" from any open ssh session), making spoofing of both checksum and the
original file a bit harder.



### Content

##### pysed

This one is for simple pcre-based text replacement, basically a sed's
"s/from/to/" command with lookahead/lookbehind assertions.

Example, to replace all two-space indents with tabs and drop space-based inline
alignment:

	pysed '(?<=\w)\s+(?=\w)' ' ' '^\s*  ' '\t' -i10 -b somecode.py

##### pysort

Sort file contents, based on some key-field with support for multiple field
delimeters. Wrote this one when my /etc/passwd got messy and I just wanted to
sort its contents by uid.

##### color

Outputs terminal color sequences, making important output more distinctive.

Also can be used to interleave "tail -f" of several logfiles in the same terminal:

	t -f /var/log/app1.log | color red - &
	t -f /var/log/app2.log | color green - &
	t -f /var/log/app2.log | color blue - &

##### znc-log-aggregator

Tool to process znc chat logs, produced by "log" module (global, per-user or
per-network - looks everywhere) and store them using following schema:

	<net>/chat/<channel>__<yy>-<mm>.log.xz
	<net>/priv/<nick>__<yy>-<mm>.log.xz

Where "priv" differs from "chat" in latter being prefixed by "#" or "&".
Values there are parsed according to any one of these (whichever matches first):

 - `users/<net>/moddata/log/<chan>_<date>.log`
 - `moddata/log/<net>_default_<chan>_<date>.log` (no "_" in `<net>` allowed)
 - `moddata/log/<user>_<net>_<chan>_<date>.log` (no "_" in `<user>` or `<net>` allowed)

Each line gets processed by regexp to do `[HH:MM:SS] <nick> some msg` ->
`[yy-mm-dd HH:MM:SS] <nick> some msg`.

Latest (current day) logs are skipped.
New logs for each run are concatenated to the monthly .xz file.

Should be safe to stop at any time without any data loss - all the resulting
.xz's get written to temporary files and renamed at the very end (followed only
by unlinking of the source files).

All temp files are produced in the destination dir and should be cleaned-up on
any abort/exit/finish.

Idea is to have more convenient hierarchy and less files for easier shell
navigation/grepping (xzless/xzgrep), plus don't worry about the excessive space
usage in the long run.

##### znc-log-reader

Same as znc-log-aggregator above, but seeks/reads specific tail ("last n lines")
or time range (with additional filtering by channel/nick and network) from all
the current and aggregated logs.

##### resolve-hostnames

Scrpt to find all the specified (either directly, or by regexp) hostnames and
replace these with corresponding IP addresses (resolved through getaddrinfo).

Examples:

	% cat cjdroute.conf
	... "fraggod.net:21987": { ... },
		"localhost:21987": { ... },
		"fraggod.net:12345": { ... }, ...

	% resolve-hostnames fraggod.net < cjdroute.conf
	... "192.168.0.11:21987": { ... },
		"localhost:21987": { ... },
		"192.168.0.11:12345": { ... }, ...

	% resolve-hostnames fraggod.net localhost < cjdroute.conf
	... "192.168.0.11:21987": { ... },
		"127.0.0.1:21987": { ... },
		"192.168.0.11:12345": { ... }, ...

	% resolve-hostnames -m '"(?P<name>[\w.]+):\d+"' < cjdroute.conf
	... "192.168.0.11:21987": { ... },
		"127.0.0.1:21987": { ... },
		"192.168.0.11:12345": { ... }, ...

	% resolve-hostnames fraggod.net:12345 < cjdroute.conf
	... "fraggod.net:21987": { ... },
		"localhost:21987": { ... },
		"192.168.0.11:12345": { ... }, ...

	% resolve-hostnames -a inet6 fraggod.net localhost < cjdroute.conf
	... "2001:470:1f0b:11de::1:21987": { ... },
		"::1:21987": { ... },
		"2001:470:1f0b:11de::1:12345": { ... }, ...

Useful for tools that cannot or should not handle names or to just convert lists
of names (in some arbitrary format) to IP addresses.
Has all sorts of failure-handling and getaddrinfo-control cli options, can
resolve port/protocol names as well.

##### pcap-process

Wrapper around tshark to run it (with specified parameters) on a pcap file and
pick stuff (via specified on command line selectors) from its "pdml" (xml)
output.

Kinda same as `tshark -T fields`, but with python on top.

For example, `pcap-process -f http.request.uri/show dump.pcap` will print full
parsed values (as presented by tshark dissector) of "http.request.uri" fields.

##### hype

Tools to work with [cjdns](https://github.com/cjdelisle/cjdns/) and
[Hyperboria](http://hyperboria.net/) stuff.

Has lots of subcommands for cjdns admin interface interaction, various related
data processing, manipulation (ipv6, public key, switchLabel, config file,
etc) and obfuscation.
Full list with descriptions and all possible options is in --help output.

Some of the functionality bits are described below.

-- Subcommand: decode-path

Decode cjdns "Path" to a sequence of integer "peer indexes", one for each hop.

Relies on encoding schema described in NumberCompress.h of cjdns.
Nodes are not required to use it in theory, and there are other encoding schemas
implemented which should break this tool's operation, but in practice no one
bothers to change that default.

Example: hype decode-path 0000.013c.bed9.5363 -> 3 54 42 54 15 5 30

-- Subcommand: conf-paste

Obfuscates cjdns config file (cjdroute.conf) in a secure and (optionally)
deterministic way.

Should be useful to pastebin your config file without revealing most sensitive
data (passwords and keys) in it.
Might still reveal some peer info like IP endpoints, contacts, comments, general
list of nodes you're peered with. Use with caution.

Sensitive bits are regexp-matched (by their key) and then value is processed
through pbkdf2-sha256 and output is truncated to appear less massive.
pbkdf2 parameters are configurable (see --help output), and at least
--pbkdf2-salt should be passed for output to be deterministic, otherwise random
salt value will be used.



### Dev

##### tabs_filter

My secret weapon in tabs-vs-spaces holywar.

In my emacs, tab key always inserts "\t", marking spaces as a bug with
develock-mode.
This script transparently converts all indent-tabs into spaces and back,
designed to be used from git content filters, and occasionally by hand.

.git/config:

	[filter "tabs"]
		clean = tabs_filter clean %f
		smudge = tabs_filter smudge %f

.git/info/attributes or .gitattributes:

	*.py filter=tabs
	*.tac filter=tabs

Not sure why people have such strong opinions on that trivial matter, but I find
it easier never to mention that I use such script ;)

##### golang_filter

Same idea as in "tabs_filter", but on a larger scale - basically does to
[Go](http://golang.org/) what
[coffee-script](http://jashkenas.github.com/coffee-script/) does to the syntax
of javascript - drops all the unnecessary brace-cancer, with the ability to
restore original perfectly ("diff -u reverse original" is checked upon
transformation to make sure of that), as long as code intentation is correct.

.git/config:

	[filter "golang"]
		clean = golang_filter git-clean %f
		smudge = golang_filter git-smudge %f

.git/info/attributes or .gitattributes:

	*.go filter=golang

Again, ideally no one should even notice that I actually don't have that crap in
the editor, while repo and compiler will see the proper (bloated) code.

##### distribute_regen

Tool to auto-update python package metadata in setup.py and README files.

Uses python ast module to parse setup.py to find "version" keyword there and
update it (via simple regex replacement, not sure if ast can be converted back
to code properly), based on date and current git revision number, producing
something like "12.04.58" (year.month.revision-since-month-start).

Also generates (and checks with docutils afterwards) README.txt (ReST) from
README.md (Markdown) with pandoc, if both are present and there's no README or
README.rst.

Designed to be used from pre-commit hook, like `ln -s /path/to/distribute_regen
.git/hooks/pre-commit`, to update version number before every commit.

##### darcs_bundle_to_diff

Ad-hoc tool to dissect and convert darcs bundles into a sequence of unified diff
hunks. Handles file creations and all sorts of updates, but probably not moves
and removals, which were outside my use-case at the moment.

Was written for just one occasion (re-working old bundles attached to tahoe-lafs
tickets, which crashed darcs on "darcs apply"), so might be incomplete and a bit
out-of-date, but I imagine it shouldn't take much effort to make it work with
any other bundles.

##### git-nym

Script to read NYM env var and run git using that ssh id instead of whatever
ssh-agent or e.g. `~/.ssh/id_rsa` provides.

NYM var is checked for either full path to the key, basename in `~/.ssh`, name
like `~/.ssh/id_{rsa,ecdsa,ed25519}__${NYM}` or unique (i.e. two matches will
cause error, not random pick) match for one of `~/.ssh/id_*` name part.

Can be used as `NYM=project-x git-nym clone git@dev.project-x:component-y` to
e.g. clone the specified repo using `~/.ssh/id_rsa__project-x` key or as
`NYM=project-x git nym clone ...`.
Also to just test new keys with git, disregarding ssh-agent and lingering
control sockets with NYM_CLEAN flag set.

##### git-meld

Git-command replacement for git-diff to run meld instead of regular
(git-provided) textual diff, but aggregating all the files into one invocation.

For instance, if diffs are in `server.py` and `client.py` files, running `git
meld` will run something like:

	meld \
		--diff /tmp/.git-meld/server.py.hash1 /tmp/.git-meld/server.py.hash2 \
		--diff /tmp/.git-meld/client.py.hash1 /tmp/.git-meld/client.py.hash2

Point is to have all these diffs in meld tabs (with one window per `git meld`)
instead of running separate meld window/tab on each pair of files as setting
GIT_EXTERNAL_DIFF would do.

Should be installed as `git-meld` somewhere in PATH *and* symlinked as
`meld-git` (git-meld runs `GIT_EXTERNAL_DIFF=meld-git git diff "$@"`) to work.

##### catn

Similar to "cat" (specifically coreutils' `cat -n file`), but shows specific
line in a file with a few "context" lines around it:

	% catn js/main.js 188
	   185:     projectionTween = function(projection0, projection1) {
	   186:       return function(d) {
	   187:         var project, projection, t;
	>> 188:         project = function(λ, φ) {
	   189:           var p0, p1, _ref1;
	   190:           λ *= 180 / Math.PI;
	   191:           φ *= 180 / Math.PI;

Above command is synonymous to `catn js/main.js 188 3`, `catn js/main.js:188`
and `catn js/main.js:188:3`, where "3" means "3 lines of context" (can be
omitted as 3 is the default value there).

`catn -q ...` outputs line + context verbatim, so it'd be more useful for piping
to another file/command or terminal copy-paste.

##### git_terminate

Script to permanently delete files/folders from repository and its history -
including "dangling" objects where these might still exist.

Should be used from repo root with a list of paths to delete,
e.g. `git_terminate path1 path2`.

WARNING: will do things like `git reflog expire` and `git gc` with agressive
parameters on the whole repository, so any other possible history not stashed or
linked to existing branches/remotes (e.g. stuff in `git reflog`) will be purged.

##### git_contains

Checks if passed tree-ish (hash, trimmed hash, branch name, etc - see
"SPECIFYING REVISIONS" in git-rev-parse(1)) object(s) exist (e.g. merged) in a
specified git repo/tree-ish.

Essentially does `git rev-list <tree-ish2> | grep $(git rev-parse <tree-ish1>)`.

	% git_contains -C /var/src/linux-git ee0073a1e7b0ec172
	[exit status=0, hash was found]

	% git_contains -C /var/src/linux-git ee0073a1e7b0ec172 HEAD notarealthing
	Missing:
	  notarealthing
	[status=2 right when rev-parse fails before even starting rev-list]

	% git_contains -C /var/src/linux-git -H v3.5 --quiet ee0073a1e7b0ec172
	[status=2, this commit is in HEAD, but not in v3.5 (tag), --quiet doesn't produce stdout]

	% git_contains -C /var/src/linux-git --any ee0073a1e7b0ec172 notarealthing
	[status=0, ee0073a1e7b0ec172 was found, and it's enough with --any]

	% git_contains -C /var/src/linux-git --strict notarealthing
	fatal: ambiguous argument 'notarealting': unknown revision or path not in the working tree.
	Use '--' to separate paths from revisions, like this:
	'git <command> [<revision>...] -- [<file>...]'
	git rev-parse failed for tree-ish 'notarealting' (command: ['git', 'rev-parse', 'notarealting'])

Lines in square brackets above are comments, not actual output.



### Misc

##### systemd-dashboard

[There's a post](http://blog.fraggod.net/2011/2/Dashboard-for-enabled-services-in-systemd)
with more details.

	root@damnation:~# systemd-dashboard -h
	usage: systemd-dashboard [-h] [-s] [-u] [-n] [-x]

	Tool to compare the set of enabled systemd services against currently running
	ones. If started without parameters, it'll just show all the enabled services
	that should be running (Type != oneshot) yet for some reason they aren't.

	optional arguments:
	  -h, --help            show this help message and exit
	  -s, --status          Show status report on found services.
	  -u, --unknown         Show enabled but unknown (not loaded) services.
	  -n, --not-enabled     Show list of services that are running but are not
	                        enabled directly.
	  -x, --systemd-internals
	                        Dont exclude systemd internal services from the
	                        output.

	root@damnation:~# systemd-dashboard
	smartd.service
	systemd-readahead-replay.service
	apache.service

##### at

Replacement for standard unix'ish "atd" daemon in the form of a bash script.

It just forks out and waits for however long it needs before executing the given
command. Unlike with atd, such tasks won't survive reboot, obviously.

	Usage: ./at [ -h | -v ] when < sh_script
	With -v flag ./at mails script output if it's not empty even if exit code is zero.

##### mail

Simple bash wrapper for sendmail command, generating From/Date headers and
stuff, just like mailx would do, but also allowing to pass custom headers
(useful for filtering error reports by-source), which some implementations of
"mail" fail to do.

##### passgen

Uses adict english dictionaly to generate easy-to-remember passphrase.
Should be weak if bruteforce attack picks words instead of individual lettters.

##### ssh-tunnel

Script to keep persistent, unique and reasonably responsive ssh tunnel.
Mostly just a wrapper with collection of options for such use-case.

##### task

Wrapper scripts to run stuff from cron:

* Introducing controllable random delays (so the same crontab line on multiple
  servers won't be ran in sync, introducing unnecessary load spikes on any
  shared resources).
* Adding syslog entries about jobs' start/stop/delay.
* Running job from a oneshot systemd service, to enforce any arbitrary cgroup
  limits via unit file, dependencies and prevent parallel execution.

##### urlparse

Simple script to parse long URL with lots of parameters, decode and print it out
in an easily readable ordered YAML format or diff (that is, just using "diff"
command on two outputs) with another URL.

No more squinting at some huge incomprehensible ecommerce URLs before scraping
the hell out of them!

##### openssl-fingerprint

Do `openssl s_client -connect somesite </dev/null | openssl x509 -fingerprint
-noout -sha1` in a nicer way - openssl cli tool doesn't seem to have that.

Also can be passed socks proxy IP:PORT to use socat and pipe openssl connection
through it - for example, to get fingerprint over Tor (with `SocksAddress
localhost:1080`) link:

	% openssl-fingerprint google.com localhost:1080
	SHA1 Fingerprint=A8:7A:93:13:23:2E:97:4A:08:83:DD:09:C4:5F:37:D5:B7:4E:E2:D4

##### graphite-scratchpad

Tool to load/dump stored [graphite](http://graphite.readthedocs.org/) graphs
through formats easily editable by hand.

For example, creating even one dashboard there is a lot of clicky-clicks, and 10
slightly different dashboards is mission impossible, but do `graphite-scratchpad
dash:top` (loaded straight from graphite db) and you get:

    name: top

    defaultGraphParams:
      from: -24hours
      height: 250
      until: -20minutes
      width: 400

    ...

    graphs:
      - target:
          - *.memory.allocation.reclaimable
      - target:
          - *.disk.load.sdb.utilization
          - *.disk.load.sda.utilization
        yMax: 100
        yMin: 0
      - target:
          - *.cpu.all.idle
        yMax: 100
        yMin: 0
    ...

That's all graph-building data in an easily readable, editable and parseable
format (yaml, nicely-spaced with [pyaml](https://github.com/mk-fg/pretty-yaml)
module).

Edit that and do `graphite-scratchpad yaml dash:top < dash.yaml` to replace the
thing in graphite db with an updated thing.
Much easier than doing anything with GUI.

##### ip-ext

Some minor tools for network configuration from scripts, which iproute2 seem to
be lacking.

For instance, if network interface on a remote machine was (mis-)configured in
initramfs or wherever to not have link-local IPv6 address, there seem to be no
way to (re-)add it without doing "ip link down && ip link up", which is a red
flag for a remote machine over such trivial matter.

`ipv6-link-local` subcommand handles that particular case, generating
ipv6-lladdr from mac, as per RFC 4291 (as implemented in "netaddr" module) and
can assign resulting address to the interface, if missing:

```console
# ip-ext --debug ipv6-link-local -i enp0s9 -x
DEBUG:root:Got lladdr from interface (enp0s9): 00:e0:4c:c2:78:86
DEBUG:root:Assigned ipv6_lladdr (fe80::2e0:4cff:fec2:7886) to interface: enp0s9
```

`ip-check` subcommand allows to check if address (ipv4/ipv6) is assigned to any
of the interfaces and/or run "ip add" (with specified parameters) to assign it,
if not.

##### adhocapd

Picks first wireless dev from `iw dev` and runs hostapd + udhcpd (from busybox)
on it.

Use-case is plugging wifi usb dongle and creating temporary AP on it - kinda
like "tethering" functionality in Android and such.

Configuration for both is generated using reasonable defaults - distinctive
(picked from `ssid_list` at the top of the script) AP name and random password
(using `passgen` from this repo or falling back to `tr -cd '[:alnum:]'
</dev/urandom | head -c10`).

Dev, ssid, password, ip range and such can also be specified on the command line
(see --help).

If inet access thru local machine is needed, don't forget to also do something
like this (with default ip range of 10.67.35.0/24 and "wlp0s18f2u2" interface
name):

	# sysctl -w net.ipv4.conf.all.forwarding=1
	# iptables -t nat -A POSTROUTING -s 10.67.35.0/24 -j MASQUERADE
	# iptables -A FORWARD -s 10.67.35.0/24 -i wlp0s18f2u2 -j ACCEPT
	# iptables -A FORWARD -d 10.67.35.0/24 -o wlp0s18f2u2 -j ACCEPT

These rules are also echoed in the script, with IP and interface name that was
used.

For consistent naming of network interfaces from usb devices (to e.g. have
constant set of firewall rules for these), following udev rule can be used (all
usb-wlan interfaces will be named according to NAME there):

	SUBSYSTEM=="net", ACTION=="add", ENV{DEVTYPE}=="wlan",\
		DEVPATH=="*/usb[0-9]/*", NAME="wlan_usb"

##### mikrotik-backup

Script to ssh into [mikrotik router](http://mikrotik.com) with specified
("--auth-file" option) user/password and get the backup, optionally compressing
it.
Can determine address of the router on its own (using "ip route get").
Can be used more generally to get/store output of any command(s) to the router.

RouterOS allows using DSA (old, disabled on any modern sshds) keys, which should
be used if accessible at the standard places (e.g. "~/.ssh/id_dsa").
That might be preferrable to using password auth.

Python script, uses "twisted.conch" for ssh.



### Desktop

Helpers for more interactive (client) machine, DE and apps there.


#### uri_handlers

Scripts to delegate downloads from firefox to a more sensible download managers.

Mostly I use remote mldonkey for ed2k and regular http downloads and rtorrent /
transmission for bittorrent (with some processing of .torrent files to drop
long-dead trackers from there and flatten tracker tiers, for reasons I blogged
about in some distant past).


#### media

Scripts - mostly wrappers around ffmpeg and pulseaudio - to work with (process)
various media files and streams.

###### video_to_ogg

Straightforward "ffmpeg -f wav | oggenc" script to convert any media that has
sound to an ogg file.

###### flv_merge

Merges multiple segments of video downloaded from some streaming service
(e.g. twitch.tv) into one full flv via tricky ffmpeg h264/aac
[stream concatenation functionality](http://ffmpeg.org/trac/ffmpeg/wiki/How%20to%20concatenate%20%28join%2C%20merge%29%20media%20files).

###### parec_from_flash

Creates null-sink in pulseaudio and redirects browser flash plugin audio output
stream to it, also starting "parec" and oggenc to record/encode whatever happens
there.

Can be useful to convert video to podcast if downloading flv is tricky for
whatever reason.

###### pa_track_history

Queries pa sinks for specific pid (which it can start) and writes "media.name"
(usually track name) history, which can be used to record played track names
from e.g. online radio stream in player-independent fashion.

###### mpv_icy_track_history

Same as pa_track_history above, but gets tracks when [mpv](http://mpv.io/) dumps
icy-* tags (passed in shoutcast streams) to stdout, which should be at the start
of every next track.

More efficient and reliable than pa_track_history, but obviously mpv-specific.

###### totty

Wrapper around awesome [img2xterm](https://github.com/rossy2401/img2xterm) tool
to display images in a color-capable terminal (e.g. xterm).

Useful to query "which image is it" right from tty.
Quality of the resulting images is kinda amazing, given tty limitations.


#### notifications

A bunch of tools to issue various desktop notifications.

###### exec

Wrapper to run specified command and notify (via
[desktop-notifications](http://developer.gnome.org/notification-spec/) only atm)
if it fails (including "no such binary" errors) or produces any stderr.
Optionally produces notification in any case.

Useful mainly for wrapping hooks in desktop apps like firefox, to know if click
on some "magnet:..." link was successfully processed or discarded.

	% notify.exec -h --
	usage: notify.exec [ options... -- ] command [ arguments... ]

	Wrapper for command execution results notification.

	optional arguments:
	  -h, --help            show this help message and exit
	  -e, --exit-code-only  Issue notification only if exit code not equals zero,
	                        despite stderr.
	  -v, --notify-on-success
	                        Issue notification upon successful execution as well.
	  -d, --dump            Include stdou/stderr for all notifications.


###### power

Script to spam sounds and desktop-notifications upon detecting low battery level.
Not the only one to do somethng like that on my system, but saved me some work
on many occasions.

###### skype

Script to make Skype use desktop-notification protocol.

* open Skype
* open the menu and press 'Options' or press Ctrl-O
* go to 'Notifications' tab
* hit the 'Advanced' button and check 'Execute the following script on _any_ event'
* paste: notify.skype "%type" "%sname" "%fname" "%fpath" "%smessage" "%fsize" "%sskype"
* disable or enable the notifications you want to receive.

Imported from [this gist](https://gist.github.com/1958564).

###### logtail

Script to watch log files (as many as necessary) for changes with inotify and
report any new lines appearing there via desktop notifications.

Can remember last position in file either by recording it in file's xattrs or in
a shelve db (specified via -x option).
Doesn't do much with it by default though, starting to read files from the end,
but that can be fixed by passing --keep-pos.

Somewhat advanced usage example:

	notify.logtail\
	  -i ~/media/appz/icons/biohazard_48x.png\
	  -x "$XDG_RUNTIME_DIR"/notify.logtail.db\
	  /var/log/messages

###### mail

Daemon script to monitor dovecot delivery logs (either generic ones, or produced
via "mail_log" plugin), efficiently find delivered messages by their message-id
and issue desktop notification to a remote host with parsed message details
(path it was filed under, decoded from and subject headers).

Things like rsyslog make it fairly easy to create a separate log with such
notifications for just one user, e.g.:

	if (
	  $programname == 'dovecot'
	  and $syslogfacility-text == 'mail'
	  and $syslogseverity-text == 'info'
	  and re_match($msg, '^lda\\(someuser\\): sieve: msgid=[^:]+: stored mail into mailbox .*') )
	then action(
	  type="omfile" FileCreateMode="0660"
	  FileOwner="root" FileGroup="someuser"
	  File="/var/log/processing/mail.deliver.someuser.log" )

Remote notifications are delivered to desktop machines via robust zeromq pub/sub
sockets as implemented in
[notification-thing](https://github.com/mk-fg/notification-thing/#network-broadcasting)
daemon I have for that purpose.

Even idle-imap doesn't seem to provide proper push notifications with multiple
folders yet, and this simple hack doesn't even require running a mail client.


##### pick_tracks

A simple tool to randomly pick and copy files (intended usage is music tracks)
from source to destination.

Difference from "cp" is that it will stop when destination will be filled (to
the configurable --min-df threshold) and will pick files in arbitrary order from
arbitrary path hierarchy.

Use-case is simple - insert an SD card from a player and do:

	% mount /mnt/sd_card
	% rm -rf /mnt/sd_card/music
	% pick_tracks -s 200 /mnt/music/OverClocked_Remix /mnt/sd_card/music
	INFO:root:Done: 1673.1 MiB, rate: 1.29 MiB/s

"--debug" also keeps track of what's being done and calculates how much time is
left based on df-goal and median rate.

Source dir has like [3k files](http://ocremix.org/torrents/) in many dirs, and
cp/rsync will do the dumb "we'll copy same first things every time", while this
tool will create the dst path for you, copy always-new selection there and - due
to "-s 200" - leave 200 MiB there for podcasts you might want to also upload.

As with "cp", "pick_tracks /path1 /path2 /dst" is perfectly valid.

And there are neat cleaup flags for cases when I need to cram something new to
the destination, preserving as much of the stuff that's already there as
possible (and removing least important stuff).
Cleanup (if requested) also picks stuff at random up to necessary df.

"--shuffle" option allows to shuffle paths on fat by temporarily copying them
off the media to some staging area and back in random order.
Use-case is dump mp3 players that don't have that option.

Uses [plumbum](http://plumbum.readthedocs.org) to call "rsync --inplace" (faster
than "cp" in most cases) and "find" to do the actual copy/listing.

##### link

ssh wrapper, to save time on typing something like `exec ssh -X -A -p3542
root@1.2.3.4 'screen -DR'`, especially for N remote hosts.

Also has the ability to "keep trying to connect", useful (to me, at least) for
crappy shared-hosting servers, where botnets flood ssh with slowloris-like
attacks on it's authentication, exceeding limit on unauthorized connections in
sshd.

##### e_config_backup

[Yapps2](https://github.com/mk-fg/yapps)-based (grammar as-is in *.g file)
parser script for Enlightenment (E17) config file (doing eet-decoding
beforehand) for the purposes of it's backup [in
git](https://github.com/mk-fg/de-setup) alongside other DE-related
configuration.

Whole purpose of decoding/encoding dance is to sort the sections (which E orders
at random) and detect/filter-out irrelevant changes like remembered window
positions or current ([auto-rotated](http://desktop-aura.sourceforge.net/))
wallpaper path.

##### vfat_shuffler

Tool to shuffle entries inside a vfat (filesystem) directory.

Some crappy cheap mp3 players don't have shuffle functionality and play files
strictly in the same order as their [dentries](https://en.wikipedia.org/wiki/File_Allocation_Table#Directory_entry)
appear on the device blocks.

Naturally, as same order gets boring, one way to "shuffle" them is to re-upload
same files in random order (as "pick_tracks" tool does here), but if it's a few
gigs of files, such method is slow and wears-out flash drive unnecessarily.

So easy fix is just to swap dentries' places, which unfortunately requires
re-implementing a bit of vfat driver code, which (fortunately) isn't that
complex.

Tool takes a device name and directory to shuffle as arguments (see --help) and
has --list and --dry-run flags to make sure it'll do what one wants it to, and
not bug-out halfway there on parsing fs.

One limitation is that it works *only* with "vfat" fs type, which can be created
with "mkfs.vfat" tool, *not* the types that "mkdosfs" tool creates, *not* fat16
or whatever other variations are there.
Reason is just that I didn't bother to learn the differences between these, just
checked and saw parser bug out on mkdosfs-created fs format.

Might be useful baseline to hack some fat32-related tool, as it has everything
necessary for full r/w implementation - e.g. a tool to hardlink files on fat32,
create infinite dir loops, undelete tool, etc.

##### fan_control

Script to control speed of dying laptop fan on Acer S3 using direct reads/writes
from/to `/dev/ports` to not run it too fast (causing loud screech and vibrating
plastic) yet trying to keep cpu cool enough.

Or, failing that, use cpupower tool to drop frequency (making it run cooler in
general) and issue dire warnings to desktop.

##### emms_beets_enqueue

Script to query [beets](http://beets.readthedocs.org/) music database (possibly
on a remote host) with specified parameters and add found tracks to
[EMMS](https://www.gnu.org/software/emms/) playlist (via emacsclient).

Also allows to just dump resulting paths or enqueue a list of them from stdin.

##### emms_current

Dumps path of the current track into terminal.
Easy to do with emacsclient, but needs extra sed to fix the escaping that elisp does.

##### ff_backup

Script to backup various firefox settings in a diff/scm-friendly manner
(i.e. decoded from horrible one-liner json into [pyaml](https://github.com/mk-fg/pretty-yaml)),
so that they can be tracked in e.g. git.

Written out of frustration about how YouTube Center seem to loose its shit and
resets config sometimes.

Can/should be extended to all sorts of other ff/ext settings in the future - and
probably is already, see its yaml config for details.

##### bt_agent

BlueZ bluetooth authorization agent script/daemon.

Usually included into DE-specific bluetooth applet or can be used from
"bluetoothctl" shell (`agent on`), but I don't have former (plus just don't want
to rely on any DE much) and latter isn't suitable to run daemonized.

When run interactively (default), will ask permission (y/n) to authorize new
pairings and PINs for these.

With `--authorize-services` option (and optional list of bdaddrs), will allow
any paired device to (re-)connect without asking, and `--non-interactive` flag
will turn off any possible prompts, allowing to run it in the background to only
authorize paired (and/or whitelisted) devices.



### VM

Scripts to start and manage qemu/kvm based VMs I use for various dev purposes.

These include starting simple vde-based networking, syncing kernels and
initramfs images out of vms, doing suspend/resume for running vms easily, etc.

Probably exist just because I don't need anything but qemu/kvm and know these
well enough, so don't really need abstractions libvirt provides, nothing really
special.



### sysdig

Lua ["chisels"](https://github.com/draios/sysdig/wiki/Chisels%20User%20Guide)
for [sysdig tool](https://github.com/draios/sysdig/).

Basically simple scripts to filter and format data that sysdig reads or collects
in real-time for various common tasks.



### aufs

A few tools to work with a layered aufs filesystem on arm boards.

##### aubrsync

Modified script from aufs2-util.git, but standalone (with stuff from aufs.shlib
baked-in) and not failing on ro-remounts, which seem to be a common thing for
some places like /var or /home.

##### aufs_sync

Convenience wrapper around aubrsync for mounts like `none /var -o
br:/aufs/rw/var=rw:/aufs/ro/var=ro`.
Can also just list what's there to be synced with "check" command.

	Usage: aufs_sync { copy | move | check } module
	Example (flushes /var): aufs_sync move var
