# zm-archiver
A ZoneMinder video event archiver with a very flexible time-lapse feature

## Synopsis
0: This was originally taken from: https://github.com/jacomoman/zm-archiver
	Of course, I want to give credit to the original author - however, I have modified this for my own needs and configuration.  What follows is primarily the original author's notes.  The code in this branch is customized.

The original author wrote this utility because:
1. I've based my Video Recorder needs around ZoneMinder (ZM)
2. I needed a way to archive ZM saved events with more flexibility & fine-grained control

## Design
This utility parses a user-supplied archiver "spec" file and subsequently grabs the JPG files in the ZM events folder and archives them to be processed later by "ffmpeg" to create a MPEG video file.  
It does the following: 
	* copies the jpg files all into a single directory, 
	* creates a TAR image to use as input to "ffmpeg"

## Disclaimer
This is a non-distructive tool - saying that I mean that it will not remove or modify any of the original files.  If you decide to use this suite of tools, you are using this at your own risk.  It is implied that you know Linux, unix shell scripting and Zoneminder functionality prior to attmpting to use/modify this suite of scripts. 

## Copyright
Copyright (c) 2018 by Howard I Grapek <howiegrapek@yahoo.com>
All rights reserved.

## License
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:
 
    - Redistributions of source code must retain the above copyright notice, this
      list of conditions and the following disclaimer.
 
    - Redistributions in binary form must reproduce the above copyright notice,
      this list of conditions and the following disclaimer in the documentation
      and/or other materials provided with the distribution.
 
 THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
 AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
 DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
 FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
 CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
 OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
 OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

### Format of the archiver spec file:

**IMPORTANT Note**: The `path` elements below refer to the `year/month/day/hour/minute` path just under ZM's `DIR_EVENTS` settings in Options. The `zm_events_path` is assumed to be `/var/www/zm/events`. If you're using ZM v1.30.x and above, that's been changed to `/var/cache/zoneminder/events`. Please check the `zm_events_path` variable at the top of the `prep-jpegs.sh` script and set it to whatever you have in your installation.

The spec file is a `space`-delimited text file in the following format:

	<monitor-id> <zm-begin-path> <zm-end-path> <frames-to-skip>

* `monitor-id`* : self-explanatory
* `zm-begin-path`* : The begin ZM events path
* `zm-end-path`* : The end ZM events path
* `frames-to-skip` : # of JPEG frames to skip while archiving (optional). Default: 0 (no frames skipped)

**CAUTION**: I didn't put a whole lot of error-checking in the parser of this file. Make sure your spec file **STRICTLY** adheres to this format. I don't and can't guarantee any results if you write a malformed spec file.

### Example:
Let's say you have a monitor ID `3` that's been set to `Modect` and has been recording events for 6 months. You want to archive only certain portions of a few days of these events, without having to resort to ZM's own tedious UI, selecting events checkboxes, etc. You'd create a archiver spec text file, give it any name you like, with the following content:

	3 16/08/17/08/23 16/08/17/19/00 0
	3 16/08/18/13/08 16/08/18/23/00 20
	3 16/08/19/09/00 16/08/21/17/00 40

#### The lines in the above spec file have the following meaning (line by line):

	3 16/08/17/08/23 16/08/17/19/00 0
All events from monitor `3` beginning on August 17th 2016, 8:23AM, and ending with August 17th 2016 (same day), 7:00PM. No frames skipped.

	3 16/08/18/13/08 16/08/18/21/00 20
All events from monitor `3` beginning on August 18th 2016, 1:08PM, and ending with August 18th 2016 (same day), 11:00PM. Skip 20 frames at a time.

	3 16/08/19/09/00 16/08/21/17/00 40
All events from monitor `3` beginning on August 19th 2016, 9:00AM, and ending with August 21st 2016, 5:00PM. Skip 40 frames at a time.

#### Notes:
* If your begin & end paths don't actually have any recorded events in them or the paths are not found, the script safely (and quietly) skips them and moves forward onto the next elements in the path until it finds something to archive. So you can safely specify any begin & end `day/hour/minute` paths without having to worry about matching exact recorded event times.  
* Frame skipping, if above 0, always starts with the 2nd frame in the specified line. So 1st frame is always picked up, given number skipped, then next frame picked up, and given number skipped, and so on and so forth.


## Usage
This utility contains 2 main scripts:

* `prep-jpegs.sh`: This script reads your properly formatted spec file, walks the given paths, skipping any frames (if specified). It copies all found JPEG files to an output directory. When finished it creates a TAR file from that output directory, which is the input to the next script `make-mpg.sh`.
* `make-mpg.sh`: This script is basically a wrapper/pre-processor to `ffmpeg`. It un-TARs the JPEG frames prepared by `prep-jpegs.sh`, creates a `ffmpeg` compatible frame source file, and hands the processing to `ffmpeg` to generate a single video MPEG file.
 
### `prep-jpegs.sh` 

Once you've written your properly formatted spec file, you invoke `prep-jpegs.sh` as such:

	$ ./prep-jpegs.sh <path-to-spec-file> [<output-filename>]

where: 
* `path-to-spec-file`: The full path to your properly formatted archiver spec file.
* `output-filename`: (optional) Filename used for the output TAR file (minus `.tar`). If none given, the output filename will be equal to the name of the given spec file, minus the `.txt` extension, plus `.tar`. So if your given spec file path is: `~/specs/m3-2016-08-events.txt`, your default output TAR file will be: `~/output/m3-2016-08-events.tar`

**NOTE**: The output TAR file will be placed in a directory called `output` under your user home directory.

##### Example:
Let's say you wrote a spec file under your home directory `~/specs/` folder called `m3-2016-08-events.txt`. You'd invoke `prep-jpegs.sh` as such:

	$ ./prep-jpegs.sh ~/specs/m3-2016-08-events.txt

After some console output by the script, if all goes well, you'll end up with a TAR file at: `~/output/m3-2016-08-events.tar`. You're now ready to use the next script to create a MPEG video file from your prepared TAR file.

### `make-mpg.sh`
This script has a dependency on `ffmpeg`. It assumes that `ffmpeg` is an executable that's found in your Linux `$PATH` of your currently logged on user. If this is not the case, you'll have to set (and export) an environment variable `FFMPEG_EXEC` before you can use this script.

So do a `which ffmpeg` first. If the `ffmpeg` executable is found, then you can move onto the next step. But if not, you must issue the following:

	$ export FFMPEG_EXEC=<full-path-to-ffmpeg>

With your `ffmpeg` executable taken care of, and output TAR file created from the previous script, invoke the `make-mpg.sh` as such:

	$ ./make-mpg.sh <path-to-TAR-file> [<path-to-output-mpg-file>]

where: 
* `path-to-TAR-file`: Full path to the output TAR file prepared by the `prep-jpegs.sh` script.
* `path-to-output-mpg-file`: (optional) Full path to the resulting output MPEG movie file. If none given, the output MPEG filename will be equal to the name of the given TAR file, minus the `.tar` extension, plus `.mpg`. So if your given TAR file path is: `~/output/m3-2016-08-events.tar`, your default output MPEG file will be: `~/output/m3-2016-08-events.mpg`

##### Example:
Following on the previous example with the `m3-2016-08-events.tar` file created in the `~/output` directory:

	$ ./make-mpg.sh ~/output/m3-2016-08-events.tar

After ffmpeg is done with its processing, and if no other errors, you should end up with a `.mpg` file of the same name in the `~/output` directory:

	`~/output/m3-2016-08-events.mpg`

## In Closing
Hopefully this utility is useful to you. It does a lot, but I also know there's a lot of room for improvement. Feel free to send me pull requests.

This is a video clip which the original author created using this utility time-lapsing 8 hrs of work on his house, 1st day of demolition, to 25 seconds:

https://youtu.be/ckDTv3feIuE

Enjoy!
