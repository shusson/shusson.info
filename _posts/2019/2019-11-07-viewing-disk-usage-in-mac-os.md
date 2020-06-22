# Viewing disk usage in macOS

__07/11/2019__

![pie](https://imgs.xkcd.com/comics/disk_usage.png)

If you're having trouble finding out where all your disk space on macOS has gone, you may try to search the app store for "disk" utilities.
I was surprised to find out that almost every decent looking tool was a paid app (in 2019). Luckily macOS is based on unix and can use tools like [ncdu](https://dev.yorhel.nl/ncdu).

```bash
brew install ncdu
```

By default ncdu will scan the home directory, to scan the root directory (may take a couple minutes):

```bash
ncdu -x /
```

You'll see something like:

```bash
   ┌───Scanning...───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   │                                                                                                                                                                                                         │
   │ Total items: 12192487size: 290.6 GiB                                                                                                                                                                    │
   │ ...                                                                                                                                                                                                     │
   └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

```bash
--- /Users/shanehusson -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   50.2 GiB [##########] /Library
   33.5 GiB [######    ] /Music
   12.7 GiB [##        ] /src
    6.3 GiB [#         ] /.Trash
    5.0 GiB [#         ] /.npm
    1.5 GiB [          ] /Pictures
  945.0 MiB [          ] /miniconda3
  845.6 MiB [          ] /.vscode
```

Basic usage is through left, right, up, down keys.

HMMMMMMMMmmmmmm

```bash
--- /Users/shanehusson/Music/iTunes/iTunes Media/TV Shows ------------------------------------------------------------------------------------------------------------------------------------------------------
                         /..
   19.2 GiB [##########] /True Detective
   14.0 GiB [#######   ] /Game of Thrones
```

more info on the [man page](https://dev.yorhel.nl/ncdu/man).
