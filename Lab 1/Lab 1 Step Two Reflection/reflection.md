One paragraph explaining, in your own words, the difference between a Docker image and a Docker container.

A Docker image is a packaged read-only file that holds everything a program needs to run. It holds its code, tools, and settings. 
A container is what you get when that image is actually started up and running, giving the program its own space to work in on your computer. Each container runs on its own. 
This means you can start several from the same image without them actually interfering with each other. Any changes made while a container is running stay inside that container and don't touch the original image. 
This makes it easy to stop, delete, or restart containers without having to ever rebuild the image itself.
