# Playerctl

For true players only: spotify, vlc, audacious, bmp, xmms2, and others.

## About

Playerctl is a command-line utility and library for controlling media players that implement the [MPRIS](http://specifications.freedesktop.org/mpris-spec/latest/) D-Bus Interface Specification. Playerctl makes it easy to bind player actions, such as play and pause, to media keys. You can also get metadata about the playing track such as the artist and title for integration into statusline generators or other command-line tools.

For more advanced users, Playerctl provides an [introspectable](https://wiki.gnome.org/action/show/Projects/GObjectIntrospection) library available in your favorite scripting language that allows more detailed control like the ability to subscribe to media player events or get metadata such as artist and title for the playing track.

## Using the CLI

```
playerctl [--version] [--list-all] [--all-players] [--player=NAME] [--ignore-player=IGNORE] [--format=FORMAT] COMMAND
```

Here is a list of available commands:

| Command                      | Description                                                                                            |
|:----------------------------:| ------------------------------------------------------------------------------------------------------ |
| **`play`**                   | Command the player to play.                                                                            |
| **`pause`**                  | Command the player to pause                                                                            |
| **`play-pause`**             | Command the player to toggle between play/pause.                                                       |
| **`stop`**                   | Command the player to stop.                                                                            |
| **`next`**                   | Command the player to skip to the next track.                                                          |
| **`previous`**               | Command the player to skip to the previous track.                                                      |
| **`position [OFFSET][+/-]`** | Command the player to go to the position or seek forward or backward OFFSET in seconds.                |
| **`volume [LEVEL][+/-]`**    | Print or set the volume to LEVEL from 0.0 to 1.0.                                                      |
| **`status`**                 | Get the play status of the player. Either "Playing", "Paused", or "Stopped".                           |
| **`metadata [KEY...]`**      | Print the metadata for the current track. If KEY is passed, print only those values from the metadata. |
| **`open [URI]`**             | Command for the player to open a given URI. Can be either a file path or a remote URL.                 |
| **`loop [STATUS]`**          | Print or set the loop status. Either "None", "Track", or "Playlist".                                   |

### Selecting Players to Control

Without specifying any players to control, Playerctl will act on the first player it can find.

You can list the names of players that are available to control that are running on the system with `playerctl --list-all`.

If you'd only like to control certain players, you can pass the names of those players separated by commas with the `--player` flag. Playerctl will select the first instance of a player in that list that supports the command. To control all players in the list, you can use the `--all-players` flag.

Similarly, you can ignore players by passing their names with the `--ignore-player` flag.

Examples:

```bash
# Command the first instance of VLC to play
playerctl --player=vlc play

# Command all players to stop
playerctl --all-players stop

# Command VLC to go to the next track if it's running. If it's not, send the
command to Spotify.
playerctl --player=vlc,spotify next

# Get the status of the first player that is not Gwenview.
playerctl --ignore-player=Gwenview status
```

### Printing Properties and Metadata

You can pass a format string with the `--format` argument to print properties in a specific format. Pass the variable you want to print in the format string between double braces like `{{ VARIABLE }}`. The variables available are either the name of the query command, or anything in the metadata map which can be viewed with `playerctl metadata`. You can use this to integrate playerctl into a statusline generator.

For a simple "now playing" banner:

```bash
playerctl metadata --format "Now playing: {{ artist }} - {{ album }} - {{ title }}"
# prints 'Now playing: Lana Del Rey - Born To Die - Video Games'
```

Included in the template language are some helper functions for common formatting that you can call on template variables.

```bash
playerctl metadata --format "Total length: {{ duration(mpris:length) }}"
# prints 'Total length: 3:23'

playerctl position --format "At position: {{ duration(position) }}"
# prints 'At position: 1:16'

playerctl metadata --format "Artist in lowercase: {{ lc(artist) }}"
# prints 'Artist in lowercase: lana del rey'

playerctl status --format "STATUS: {{ uc(status) }}"
# prints 'STATUS: PLAYING'
```

| Function   | Argument        | Description                                              |
| ---------- | --------------- | -------------------------------------------------------- |
| `lc`       | string          | Convert the string to lowercase.                         |
| `uc`       | string          | Convert the string to uppercase.                         |
| `duration` | int             | Convert the duration to hh:mm:ss format.                 |
| `emoji`    | status, volume  | Try to convert the variable to an emoji representation.  |
| `fa`       | status, volume  | Try to convert the variable to an FontAwesome character. |

### Following changes

You can pass the `--follow` flag to query commands to block, wait for players to connect, and print the query whenever it changes. If players are passed with `--player`, players earlier in the list will be preferred in the order they appear unless `--all-players` is passed. When no player can support the query, such as when all the players exit, a newline will be printed. For example, to be notified of information about the latest currently playing track for your media players, use:

```bash
playerctl metadata --format '{{ playerName }}: {{ artist }} - {{ title }} {{ duration(position) }}|{{ duration(mpris:length) }}' --follow
```

## Using the Library

To use a scripting library, find your favorite language from [this list](https://wiki.gnome.org/Projects/GObjectIntrospection/Users) and install the bindings library. Documentation for the library is hosted [here](https://dubstepdish.com/playerctl). For examples on how to use the library, see the [examples](https://github.com/acrisci/playerctl/blob/master/examples) folder.

### Example Python Script

This example uses the [Python bindings](https://wiki.gnome.org/action/show/Projects/PyGObject).

```python
#!/usr/bin/env python3

from gi.repository import Playerctl, GLib

player = Playerctl.Player('vlc')


def on_metadata(player, metadata):
    if 'xesam:artist' in metadata.keys() and 'xesam:title' in metadata.keys():
        print('Now playing:')
        print('{artist} - {title}'.format(
            artist=metadata['xesam:artist'][0], title=metadata['xesam:title']))


def on_play(player, status):
    print('Playing at volume {}'.format(player.props.volume))


def on_pause(player, status):
    print('Paused the song: {}'.format(player.get_title()))


player.connect('status::playing', on_play)
player.connect('status::paused', on_pause)
player.connect('metadata', on_metadata)

# start playing some music
player.play()

if player.get_artist() == 'Lana Del Rey':
    # I meant some good music!
    player.next()

# wait for events
main = GLib.MainLoop()
main.run()
```

For a more complete example which is capable of listening to when players start and exit, see [player-manager.py](https://github.com/acrisci/playerctl/blob/master/examples/player-manager.py) from the official examples.

## Troubleshooting

Some players like Spotify require certain DBus environment variables to be set which are normally set within the session manager. If you're not using a session manager or it does not set these variables automatically (like `xinit`), launch your desktop environment wrapped in a `dbus-launch` command. For example, in your `.xinitrc` file, use this to start your WM:

```
exec dbus-launch --autolaunch=$(cat /var/lib/dbus/machine-id) i3
```

## Installing

First, check and see if the library is available from your package manager (if it is not, get someone to host a package for you) and also check the [releases](https://github.com/acrisci/playerctl/releases) page on github.

Using the cli and library requires [GLib](https://developer.gnome.org/glib/) (which is a dependency of almost all of these players as well, so you probably already have it). You can use the library in almost any programming language with the associated [introspection binding library](https://wiki.gnome.org/Projects/GObjectIntrospection/Users).

Additionally, you also need the following build dependencies:

[gobject-introspection](https://wiki.gnome.org/action/show/Projects/GObjectIntrospection) for building introspection data (configurable with the `introspection` meson option)

[gtk-doc](http://www.gtk.org/gtk-doc/) for building documentation (configurable with the `gtk-doc` meson option)

Fedora users also need to install `redhat-rpm-config`


To generate and build the project to contribute to development and install playerctl to `/`:

```
meson mesonbuild
sudo ninja -C mesonbuild install
```

Note that you need `meson >= 0.46.0` installed. In case your distro only has an older version of meson in its repository you can install the newest version via pip:

```
pip3 install meson
```

Also keep in mind that gtk-doc and gobject-introspection are enabled by default, you can disable them with `-Dintrospection=false` and `-Dgtk-doc=false`.

If you don't want to install playerctl to `/` you can install it elsewhere by exporting `DESTDIR` before invoking ninja, e.g.:

```
export PREFIX="/usr/local"
meson --prefix="${PREFIX}" --libdir="${PREFIX}/lib" mesonbuild
export DESTDIR="$(pwd)/install"
ninja -C mesonbuild install
```

You can use it later on by exporting the following variables:

```
export LD_LIBRARY_PATH="$DESTDIR/${PREFIX}/lib/:$LD_LIBRARY_PATH"
export GI_TYPELIB_PATH="$DESTDIR/${PREFIX}/lib/:$GI_TYPELIB_PATH"
export PATH="$DESTDIR/${PREFIX}/bin:$PATH"
```

## License

This work is available under the GNU Lesser General Public License (See COPYING).

Copyright © 2014, Tony Crisci
