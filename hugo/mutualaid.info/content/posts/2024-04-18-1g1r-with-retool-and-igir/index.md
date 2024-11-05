---
title: "Creating one game, one ROM (1G1R) sets with Retool and igir"
draft: 
---

I was frustrated recently by the limited number of games available in some of the Nintendo Switch Online Classic Games libraries, so I decided to set up [RetroPie](https://retropie.org.uk/) on an old Raspberry Pi 3 Model B+ to play some of my favorite older games. I've set up RetroPie before, but I never connected it to my TV and never used a decent controller with it (I ended up getting a couple of [8BitDo Ultimate C 2.4G controllers](https://shop.8bitdo.com/products/8bitdo-ultimate-c-2-4g), which have been great).

This post covers:

* Creating 1G1R (one game, one ROM) sets for a few consoles
* Downloading game metadata (box art, description, etc.)

The benefit of 1G1R sets is that you won't end up with multiple ROM files for the same game or ROMs for games that are in languages that you don't understand. For example, the process of creating the 1G1R set might filter out ROMs for any non-English releases of a game, or any games that don't have an English translation.

A lot of the recommended ROM managers I found only run on Windows, but this post will use tools that run on macOS and Linux. All of the commands below will assume you've set a shell variable `roms_dir` for your desired working directory. For example, to use a subdirectory `ROMs` in your home directory:

```sh
export roms_dir="${HOME}/ROMs"
```

You'll also need Python 3 (anything 3.8 or greater should be fine) and Nodejs (I usually install the latest LTS version, which is currently 20). On macOS, you can install both of these with [Homebrew](https://brew.sh/):

```sh
brew install python@3.12 nodejs@20
```

## ROMs and DAT files

The first step of this process is to get a complete set of ROMs and a DAT/data file. DAT files are XML-formatted files that end in `.dat` and include data (like game name, description, filename, and checksum) about each ROM in a set.

You can download a ROM set from [https://archive.org/details/ni-romsets](https://archive.org/details/ni-romsets) (this is the best source I've found; let me know if you know of a better, more definitive, or more up-to-date source for No-Intro sets). For the examples in this post, I'll assume you downloaded the `Atari – 2600`, `Nintendo - Super Nintendo Entertainment System`, and `Sega - Master System - Mark III` sets and extracted them to a new directory `${roms_dir}/Sets/No-Intro`.

Download the [latest No-Intro DAT files from DAT-o-MATIC](https://datomatic.no-intro.org/index.php?page=download&op=daily&s=64) to a new directory `${roms_dir}/DAT files/No-Intro`. Select the `P/C XML` option to get the DAT files that include parent/clone info, and only select the `Main` set (excluding DAT files for things like source code and unofficial games).

## Filtering with Retool

The second step is to filter the comprehensive No-Intro DAT files to DAT files that only include the ROMs that you want in your 1G1R sets. I found [Retool](https://unexpectedpanda.github.io/retool/) to be the best tool for doing this. [igir](https://igir.io/) (the ROM manager we'll use later) also has 1G1R features, but I found that it doesn't reliably choose the latest revision (even with `--prefer-revision-newer`) and doesn't filter out compilations like Mario Bros + Duck Hunt. It's appealing as a single tool to do everything, but Retool does a better job (probably at least partly because it works off of [its own clonelists](https://github.com/unexpectedpanda/retool-clonelists-metadata) and not just ROM names).

Download the [`user-config.yaml`](./files/retool/user-config.yaml) file and place it in a new directory `${roms_dir}/Configs/Retool/`. The config file has a list of manual exclusions for titles that Retool doesn't exclude with the provided filters otherwise. There are comments in the file explaining each exclusion.

Follow the [instructions to install Retool](https://unexpectedpanda.github.io/retool/download/#git-and-python-gui-and-cli), update its clone lists and metadata files, and create a directory for Retool-generated DAT files:

```sh
python3 retool.py --update
mkdir "${roms_dir}/DAT files/Retool"
```

Use Retool to generate the filtered DAT files (`-l` filters by language, `-y` prefers licensed versions, and the `--exclude` pattern excludes everything except for games; you can [read more about the command-line options here](https://unexpectedpanda.github.io/retool/how-to-use-retool-cli/)):

```sh
python3 retool.py \
    "${roms_dir}/DAT files/No-Intro/" \
    -l -y \
    --config "${roms_dir}/Configs/Retool/user-config.yaml" \
    --exclude aAbBcdDekmMopPruv \
    --output "${roms_dir}/DAT files/Retool/"
```

## Creating sets with igir

Now that you have filtered DAT files from Retool, the final step is to use the ROM manager [igir](https://igir.io/) to copy the desired ROMs from the No-Intro sets to the 1G1R output directories. We'll exclude any Markdown files, `gamelist.xml`, and the `media` directory from being cleaned from the output directories by igir (these are often used by frontends like EmulationStation, so we don't want to remove them).

[Install igir](https://igir.io/installation/) and use it to copy ROMs based on the Retool-generated DAT files to an output directory (`-D` will create subdirectories per DAT name):

```sh
mkdir "${roms_dir}/Sets/1G1R"
igir copy clean test \
    -D \
    -C \
        "${roms_dir}/Sets/1G1R/**/*.md" \
        "${roms_dir}/Sets/1G1R/**/gamelist.xml" \
        "${roms_dir}/Sets/1G1R/**/media" \
        "${roms_dir}/Sets/1G1R/**/media/**" \
        "${roms_dir}/Sets/1G1R/**/media/**/*" \
    -d "${roms_dir}/DAT files/Retool/*.dat" \
    -i "${roms_dir}/Sets/No-Intro/" \
    -o "${roms_dir}/Sets/1G1R/"
```

You can also run igir for a single system:

```sh
igir copy clean test \
    -vvv \
    -C \
        "${roms_dir}/Sets/1G1R/**/*.md" \
        "${roms_dir}/Sets/1G1R/**/gamelist.xml" \
        "${roms_dir}/Sets/1G1R/**/media" \
        "${roms_dir}/Sets/1G1R/**/media/**" \
        "${roms_dir}/Sets/1G1R/**/media/**/*" \
    -d "${roms_dir}/DAT files/Retool/Atari - 2600*.dat" \
    -i "${roms_dir}/Sets/No-Intro/Atari - 2600" \
    -o "${roms_dir}/Sets/1G1R/Atari - 2600 (Retool)"
```

You can read more about the command-line options by running `igir --help` or [reading the igir documentation](https://igir.io/).

## Optional: Scraping metadata and generating game lists

At this point, you should have nicely curated 1G1R sets in `${roms_dir}/Sets/1G1R`. You may also want to download game metadata like box art to display in your emulation frontend. This optional step will cover how to do this using [the fork of Skyscraper used by RetroPie](https://github.com/Gemba/skyscraper). The best source I found for metadata is [ScreenScraper](https://screenscraper.fr). Depending on how large your 1G1R sets are (or how much patience you have) you may want to register an account (or even donate) to increase your API quota.

Download the [`config.ini`](files/skyscraper/config.ini) and [`artwork.xml`](files/skyscraper/artwork.xml) files for Skyscraper and place them in a new directory `${roms_dir}Configs/Skyscraper/`. In `config.ini` fill in your ScreenScraper credentials in the `userCreds` value (if you registered an account) and update any paths with your ROMS directory (replace `/Volumes/ROMs` with your value of `$roms_dir`).

Refresh the Skyscraper cache for the systems you created 1G1R sets for. This stores game metadata in the Skyscraper cache directory prior to copying it to its final destination that will be referenced from any game list files you generate.

```sh
skyscraper -c "${roms_dir}/Configs/Skyscraper/config.ini" -p atari2600 -s screenscraper
skyscraper -c "${roms_dir}/Configs/Skyscraper/config.ini" -p mastersystem -s screenscraper
skyscraper -c "${roms_dir}/Configs/Skyscraper/config.ini" -p snes -s screenscraper
```

Sometimes games aren't found on ScreenScraper because of slightly differing filenames or other matching issues. You can use Skyscraper to find which specific game metadata is missing for a console. For example, this command will look for missing metadata for the Master System ROM set:

```sh
skyscraper -c "${roms_dir}/Configs/Skyscraper/config.ini" -p mastersystem -s screenscraper --cache report:missing=all
```

Skyscraper also uses checksum matching, but this only works if the ROMs unpacked/unzipped first. I'm not sure why the game names from No-Intro and the filenames in ScreenScraper don't match sometimes, but I've found that telling Skyscraper to unpack the ROM file works to find a match in most cases when filename matching fails. For example, this command will unpack the specified Winter Olympics ROM in order to succesfully match it with the entry on ScreenScraper:

```sh
skyscraper \
    -p mastersystem \
    -s screenscraper \
    --verbosity 3 \
    --refresh \
    --flags unpack \
    "${roms_dir}/Sets/1G1R/Sega - Master System - Mark III (Retool)/Winter Olympics (Europe) (En,Fr,De,Es,It,Pt,Sv,No).zip"
```

(You can also add a pattern to Retool's `user-config.yaml` to exclude a game from the 1G1R set, and re-run Retool to remove the game (assuming you don't want the game to be a part of your 1G1R set).)

Once you've downloaded metadata for all of your games to the Skyscraper cache, you can use it to generate a game list XML file for each system.

```sh
skyscraper -c "${roms_dir}/Configs/Skyscraper/config.ini" -p atari2600
skyscraper -c "${roms_dir}/Configs/Skyscraper/config.ini" -p mastersystem
skyscraper -c "${roms_dir}/Configs/Skyscraper/config.ini" -p snes
```

That's it! You should have a curated 1G1R set of ROMs for each system, a `gamelist.xml` file in each ROM set directory, and a `media` directory with game metadata.

## Bonus

Check out [my EmulationStation theme Alcamoth](https://github.com/brianreumere/es-theme-alcamoth)! It displays both box art and a screenshot or video in the detailed view for a game, and looks pretty nice (I think).

<center>{{< figure src="images/screenshot-alcamoth-detailed.png" link="images/screenshot-alcamoth-detailed.png" alt="Screenshot of the EmulationStation detailed view showing the Game Boy game Kirby's Dream Land" width="300">}}</center>

<center>{{< figure src="images/screenshot-alcamoth-system.png" link="images/screenshot-alcamoth-system.png" alt="Screenshot of the EmulationStation system view showing the Game Gear handheld console" width="300">}}</center>
