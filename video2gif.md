

2 step process (for better quality)

Step 1.
```
ffmpeg -i input_video.webm -vf "fps=10,scale=720:-1:flags=lanczos,palettegen" palette.png
```

Step 2.
```
ffmpeg -i input_video.webm -i palette.png -filter_complex "fps=10,scale=720:-1:flags=lanczos[x];[x][1:v]paletteuse" output.gif
```

arguments
- `scale`: re-scale input video without changing aspect ratio
- `fps`: fps for output gif


## Trim GIF
1. 
```bash
ffmpeg -i input.gif -vf "palettegen" palette.png
```
2. 
```bash
ffmpeg -i input.gif -i palette.png -filter_complex "[0:v]trim=start=1:end=5,setpts=PTS-STARTPTS[p];[p][1:v]paletteuse" output.gif
```
arguments
- `start`: start time in seconds
- `end`: end time in seconds

