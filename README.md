# Guide: Google Chrome Remote Desktop Persistent Session

Ubuntu supports multiple display sessions, and Chrome Remote Desktop will (by default) leverage this feature. That means you can be connected on the machine itself, and have several applications open; when you connect over remote desktop, it will start a new session (without your existing state). Conversely, if you start doing something remotely, then try to finish it up on the machine locally, all the apps you had open won't appear on the local display. As well as being a bit annoying, this can cause all sorts of nasty bugs (e.g the most recent state in one session clobbering the other during shutdown; launching applications in one session and they actually appear in the other... it's a real mess). Follow these steps to override the "smart" functionality, and just have a single session that's shared between local and remote access.

*There are probably some very clever reasons to run it the default way, and changing it like this is less secure - for example, if you unlock the machine remotely over RDP, the machine unlocks on the local session too - someone with physical access could see your mouse moving around, watch what you were typing or even take over with a keyboard / mouse. That said, I just can't get it to work at all in the default setup, so "less secure and working" is good enough for me.*.

### Stop Chrome Remote Desktop (It is OK if it says that the daemon was not currently running)

```sh
sudo systemctl stop chrome-remote-desktop@$USER.service
```

NOTE: You might have to replace "$USER" if it spits out an error. Try pressing "tab" after "... chrome-remote-desktop..." to see what it autofills.

### Backup the original configuration

```sh
sudo cp /opt/google/chrome-remote-desktop/chrome-remote-desktop /opt/google/chrome-remote-desktop/chrome-remote-desktop.original
```

### Edit the config file

```sh
code -a /opt/google/chrome-remote-desktop/chrome-remote-desktop
```

Alternatively use nano:

```sh
sudo nano /opt/google/chrome-remote-desktop/chrome-remote-desktop
```

### Find DEFAULT_SIZES and amend to the remote computer's resolution. Example:

Found around line 78.

```python
DEFAULT_SIZES = "1920x1080"
```

Qucik tip: Use ctrl + w in nano or ctrl-f in code to find the desired variable.

Some common resolutions: ``3840x2160, 2560x1440, 1920x1080, 1280x720``


### Find the local display number in a new terminal:

```sh
echo $DISPLAY
```

### Set the X display number to the current display number (that you found in the previous command, mine was :0)

Found around line 111.

```python
FIRST_X_DISPLAY_NUMBER = 0
```

### Comment out sections that look for additional displays:

Found around line 1031.

```python
# while os.path.exists(X_LOCK_FILE_TEMPLATE % display):
# display += 1
```

### Reuse the existing X session instead of launching a new one. Alter launch_session() by commenting out launch_x_server() and launch_x_session() and instead setting the display environment variable, so that the function definition ultimately looks like the following:

Found around line 517.

```python
def launch_session(self, server_args, backoff_time):
    """Launches process required for session and records the backoff time
    for inhibitors so that process restarts are not attempted again until
    that time has passed."""
    logging.info("Setting up and launching session")
    self._init_child_env()
    self.setup_audio()
    self._setup_gnubby()
    # self._launch_server(server_args)
    # if not self._launch_pre_session():
      # If there was no pre-session script, launch the session immediately.
      # self.launch_desktop_session()
    display = self.get_unused_display_number()
    self.child_env["DISPLAY"] = ":%d" % display
    self.server_inhibitor.record_started(MINIMUM_PROCESS_LIFETIME,
                                      backoff_time)
    self.session_inhibitor.record_started(MINIMUM_PROCESS_LIFETIME,
                                     backoff_time)
```

Save and exit the editor.

### Start Chrome Remote Desktop:

```sh
sudo systemctl start chrome-remote-desktop@$USER.service
```

Please restart your computer if you still get a black screen or session select after applying the patch.

On a seperate computer, login to the remote desktop. If you have the host machine hooked up to a monitor, you should be seeing that the remote session is controlling what ever's on the screen of the local monitor.

## Known Issues

There is a more than zero chance that the Chrome Remote Desktop configuration file will reset when Chrome Remote Desktop is updated. If you are greeted with a black screen or a black screen with a session select window. Immediatly exit out of Chrome Remote Desktop and apply the solution again to the machine.

Computer might require a restart for the patch to take effect.

## Future Work

* Create a script that does the above automatically

## Resources

Look at commit history if running an older Chrome Remote desktop Version

Tested with Chrome Remote Desktop version 103.0.5060.46 and Ubuntu version 20.04
