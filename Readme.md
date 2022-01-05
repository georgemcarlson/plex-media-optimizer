# DVD Upscale

## Purpose
The DVD upscaler is designed to re-encode videos to 720p.

## Example Usage
```sh
go run cmd/client/* -h
go run cmd/client/* -path="[Path]"
go run cmd/client/* -dryRun -path="[Path]"
go run cmd/client/* -skipCleanup -skipDenoise -path="[Path]"
```

## Flags
```
  -bitrate int
    	Maximum bitrate of the resulting video. (default 1950000)
  -dryRun
    	Supply this flag when the video encoding step should be skipped.
  -filter string
    	A regex value. Only scan movies whose title matches this value. (default ".*")
  -force8Bit
    	Supply this flag when the resulting video's color depth should be 8-bit instead of 10-bit.
  -forceAvc
    	Supply this flag when the resulting video's codec should be AVC instead of HEVC.
  -path string
    	The path to the directory to scan. (default "unknown")
  -preset string
    	The preset to use. Slower preset values will produce better video quality. Valid preset values are: ultrafast superfast veryfast faster fast medium slow slower veryslow placebo  (default "slow")
  -skipCleanup
    	Supply this flag when the original videos should not be discarded.
  -skipCrop
    	Supply this flag when letter-box bars in the source video should not be removed.
  -skipDecomb
    	Supply this flag when interlaced video should not be converted to progressive video.
  -skipDenoise
    	Supply this flag when the denoiser should not be used before scaling the video.
  -skipNnedi
    	Supply this flag when the nnedi upscaler not be used to scale the video.
```

## How Optimizing Movies Works

This application will scan the supplied directory path for subdirectories that contain a movie file whose name (less the media extension) exactly maches the subdirectory name. When found, the movie file will be copied to a local temporary directory, analyzed, and (if necessory) re-encoded.

In the case that the subdirectory does not contain a movie whose name exactly matches the subdirectory name the DVD upscaler will search the subdirectory for a movie file whose name (less the media extension) ends with `- pt1`. If a match is found the DVD upscaler will go into concatination mode which means that it will concatinate all videos in the subdirectory that end in ` - pt1` through ` - pt9000` into a single movie. Chapters names based on the concatinated movies will be added to the final movie to provide a convenient way to jump to the start of a specific concatinated video. The final video will then be copied to a local temporary directory, analyzed, and (if necessory) re-encoded.

Because all media is copied to a local temporary directory the media optimizer is able to optimize remote directories that are mounted to your local filesystem. Thus, you can use rclone.org to virturaly mount your remote cloud storage system to your local file system and then supply the path to this virtural mount to the media optimizer to optimize all the movies.

## FAQ

### What is server transcoding?
Server transcoding is when the media server that is hosting the file re-encodes the video on the fly while streaming the it to the client.

### Why avoid server transcoding?
Re-encoding a video file on the fly prevents more complex video codecs (such as HEVC), scaling algorithms (such as nnedi), and denoisers (such as nlmeans) from being used effectively. Primitive scaling algorithms are undesirable because they produce a visual that has jaggier lines and more banding in gradients. Similarily, primitive denoisers are undesirable because they either cause the visual to look plasticy or retain more noise; th elater requiring a higher bitrate to encode. Lastly, the older AVC video codec will need to be used or the HEVC video will need to be used with less advanced settings. Both options will require a higher bitrate to encode the video which causes the visual that is transcoded on the fly looks blocky because the required bitrate to achieve visual transparency is so much higher then the bitrate cap.

### What typically causes server transcoding?
The server will trascode the video when the stored video file's resolution or maximum bitrate is higher than the server's or client's configured maximums. Additionally, the server will transcode the video when client's hardware does not support playback of the stored video's codec or bit-depth.

### Why set CRF to 16?
It is widely accepted that in most situations a video re-encoded using a CRF value of 16 is precieved as visually lossless compared to its source. Additionally, in most cases, the resulting average bitrate of a DVD re-encoded using the HEVC codec at a CRF of 16 will be under 2000kbps.

### Why cap bitrate to 2000kbps?
A couple of factors played into this decision. One reason is that many client video players default to a 2000kbps bandwidth limit. Staying underneath this limit will allow the media server to stream these movies to the client without first transcoding the movie to a lower bitrate. Another reason is that most DVD movies when re-encoded using the HEVC codec at a CRF of 16 will have an average bitrate under 2000kbps. This means that the bitrate cap will only be applied during the occasional high-movement scenes.

### Why scale all movies to 720p?
Many media servers and client video players default to a maximum resolution of 720p. Thus, we want to keep the maximum resolution to 720p or less to avoid server transcoding. Aditionally, proactively upscaling lower resolution videos to 720p allows us to use more advanced scaling algorithms such as nnedi which is slow and takes a ton of CPU cycles but also produces a noticably superior visual.

### Why use 10-bit color depth?
When the source video has an 8-bit color depth re-encoding the video using a 10-bit color depth lets us achieve the same visual result for about 5% less bitrate. This lowers the chance that the re-encoding the video will hit the 2000kbps bitrate cap. Additionally, using the 10-bit color depth will prevent banding when the source material is 10-bit or the upscale algorithm and filters combine to produce a gradual gradient.

### Why use the HEVC codec?
The HEVC codec is about 40% more efficient than the AVC codec which allows us to re-encode DVD videos using a CRF value of 16 to produce a video that in most situations has a bitrate less than 2000kbps. Additioanlly, most consumer video decoders support the HEVC codec with a 10-bit color depth but only support the AVC codec with an 8-bit color depth. Thus, a Plex server is able to direct stream a 10-bit HEVC video to the Plex client but most transcode the 8-bit AVC video while streaming the video to the Plex client.

### Why use denoisers ans sharpeners?
Strategic use of a denoiser allows us to dictate to the video codec what visuals to spend bitrate on by pro-actively removing video noise from the source before encoding. NLMeans is a fairly powerful denoiser that was chosen as the main denoiser to use before scaling. After scaling hqdn3d (a lightweight denoiser) and unsharp (an image sharpener) are very lightly used to remove temporal jitters that were introduced from the scaling process.

### Why crop the video?
Cropping black bars from the video allows us to proactively scale the video to take advantage of the resolution that the black bars were occupying. This extra resolution helps the visual to avoid unnecessary jaggy lines and banding.
