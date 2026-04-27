## PSP Sound build project
After much time, it was discovered that PSP & PS2 use the same sound format for the  `sfx` sounds.
See the [Sound File Analysis](https://github.com/BAD-AL/SWBF2_Xbox_mod_effort/wiki/Sound-File-Analysis) for more details.

The major difference is that PSP sound _streams_ use `altrac 3 plus` for their encoding which the `bf2 modtools` do not produce.
But since most of the PSP world 'sound' .lvl files do not contain streams, we're in a bit of luck because the modtools do include 
the ability to produce the `VAG` output consumed by both the PS2 and PSP. 
This project includes the sound config files that have the same sample rates and aliases from the PSP build.


## Usage
To build sound files for the PSP you should do:
1. The usual creation of a mod folder as described by the BF2 `modtools`.
2. You then need to download the Modder's BF sound environment
   from url: https://www.moddb.com/games/star-wars-battlefront/downloads/swbf1-swbf2-soundenv-for-modders
4. Copy the BF2 sound files to your newly created mod folder (Both BF1 and BF2 sound envs are in the download package, only copy over the stuff for BF2).
5. Copy the 'Sound' folder release from this project to your mod folder (this will overwrite the sound config files from the moddb downloaded package).
6. Ensure your mod folder is setup to munge for console (you can run the included `_BUILD\xbox_ps2_setup.bat` if needed).
7. Go the the '_BUILD' folder in your mod and open up a command line, Enter the command:

`munge.bat /platform PS2 /SOUND`

This starts the build process which will take a while. Once the build is complete, the new sound files are in your mod folder at `_LVL_PS2\\Sound`.

The various sound .lvl files to need to be checked, if you find that a sound lvl goes 'quite' you'll likely need to reduce the size by aliasing more sounds or reducing the sample rate of some sounds. 

### Stream Notes
It is possible to create valid sound streams for the PSP using the `at3tool.exe` program. 
An experimental release (v1.1) now does support building sound 'streams' for PSP. 

