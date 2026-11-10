# References

This reference would not exist without the work of these two projects. Many thanks to their authors and
contributors.

- **fefw-editor**, a Python save editor for Fire Emblem: Fortune's Weave:
  https://github.com/Antiadvocate/fefw-editor

  Its save reading and writing code was a valuable cross-check while mapping the container, the object tree and the
  unit records.

- **fefw**, a decompilation of Fire Emblem: Fortune's Weave:
  https://github.com/vexcited/fefw

  Many runtime field names and class names used in these documents come from the decompilation, and it was a great
  help in locating the game's serializers. Note that it targets a different build than update 131072, so the
  addresses in these documents (`main+0x...`) refer to the update 131072 executable, not to the decompilation.

Everything in these documents was re-checked against the update 131072 executable and against real save files.
Any remaining errors are ours.
