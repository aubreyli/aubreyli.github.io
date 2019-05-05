---
layout: post
#comments: true
categories: linux
---

## Ubuntu Tips
On ubuntu 18.04, to disable GUI on boot, run:

	sudo systemctl set-default multi-user.target

To enable GUI again issue the command:

	sudo systemctl set-default graphical.target
