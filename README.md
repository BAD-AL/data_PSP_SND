
## PSP Sound build project
After much time, it was discovered that PSP & PS2 use the same sound format for sound effects (`.sfx` files) (`vag` audio format).
See the [Sound File Analysis](https://github.com/BAD-AL/SWBF2_Xbox_mod_effort/wiki/Sound-File-Analysis) for more details.

* _Streams_ - longer sounds streamed from disk (like music and ambient sound).
* _Sound effects_ - Shorter sounds (like blaster fire and grenades).

The major difference is that PSP sound _streams_ use `altrac 3 plus` for their encoding which the `bf2 modtools` do not produce.
Since most of the PSP world 'sound' .lvl files do not contain streams, we can build PSP sound lvls for the 'worlds' with only 
modifying the `.sfx` files by changing the sample rates and aliasing more sounds. Which is the solution of our [v1.0 release](https://github.com/Gametoast/data_PSP_SND/releases/tag/1.0)

For the sound streams, we need to lean on the at3tool.exe from Sony which you can easily find via web search. This at3tool.exe program
in conjunction with the included bf_sound_tool.exe (and a slightly modified bat file) are used by the [v1.1 release](https://github.com/Gametoast/data_PSP_SND/releases/tag/1.1)
to provide a full sound munge solution for PSP (_the 1.1 release is considered `experimental` because of it's low maturity_).

## Usage
To build sound files for the PSP you should do:
1. The usual creation of a mod folder as described by the BF2 `modtools`.
2. You then need to download the Modder's BF sound environment
   from url: https://www.moddb.com/games/star-wars-battlefront/downloads/swbf1-swbf2-soundenv-for-modders
4. Copy the BF2 sound files to your newly created mod folder (both BF1 and BF2 sound envs are in the download package, only copy over the stuff for BF2).
5. Copy the 'Sound' folder release from this project to your mod folder (this will overwrite the sound config files from the moddb downloaded package).
6. Ensure your mod folder is setup to munge for console (you can run the included `_BUILD\xbox_ps2_setup.bat` if needed).
7. Go the the '_BUILD' folder in your mod and open up a command line, Enter the command:

`_BUILD>` ` munge.bat /platform PS2 /SOUND`

This starts the build process which will take a while. Once the build is complete, the new sound files are in your mod folder at `_LVL_PS2\\Sound`.

When testing your sound files, if you find that a sound lvl goes 'quite' you'll likely need to reduce the size by aliasing more sounds or reducing the sample rate of some sounds. 

### Sound Swapper Web App
There is also a web app available (_uses the bf_sound_tool library_) that will allow you to 'swap' out sounds from the sound .lvl files.
The PSP .sfx (sound effects) are swappable from .wav files.
For the PSP streams, you'll need to convert them to at3plus with `at3tool.exe` first.

(click to visit)
[![Alt Text](https://github.com/BAD-AL/bf_sound_swapper/blob/main/sound_swapper.png)](https://bad-al.github.io/bf_sound_swapper/)
