---
layout: post
title: "Self-hosting Overleaf"
date: 2025-12-01 11:45:56 +08:00
categories: tutorial overleaf latex
published: true
---

My favorite way to write reports is using LaTeX. I find most interfaces for it kind of awful, though, so when I originally found out that overleaf support a free and open source [toolkit](https://github.com/overleaf/toolkit), I immediately got to work setting it up.

# Installing the toolkit
Start simply by cloning the repo:
```bash
git clone https://github.com/overleaf/toolkit.git
cd toolkit
```

Now you need to set up your initial configuration files:
```bash
bin/init
```

Modify overleaf.rc to your liking:
```bash
# Change the IP and port to whatever you like :)
sed -i 's|^\(OVERLEAF_LISTEN_IP=\).*|\1192.168.90.98|' config/overleaf.rc
sed -i 's|^\(OVERLEAF_PORT=\).*|\18504|' config/overleaf.rc
sed -i 's|^\(SIBLING_CONTAINERS_ENABLED=\).*|\1false|' config/overleaf.rc
```

Now take note of the files available in the `config/` directory. The `version` file refers to the version of the sharelatex image being used for running the toolkit. We'll make use of this file further down.

Now start overleaf, this will download the sharelatex container, mongo and redis:
```bash
bin/up
```

- Make an admin account by navigating to 192.168.90.98:8504/launchpad
- Create a new project called test, and try to add the line:
```latex
\usepackage{glossaries-extra}
```
- Hit compile, and watch the compilation fail

The reason the compilation fails is that the overleaf-toolkit doesn't contain all the latex packages from CTAN by default. We'll need to create a custom image if want custom functionality. For example, I use glossaries-extra with bib2gls so that I can store all of my glossary entries inside of a `.bib` file, making them easier to manage.

# Creating a custom image
For now, shut down overleaf with an interrupt (ctrl-c)

Let's delete the container we initially created and reopen it:
```bash
bin/docker-compose down
bin/up -d
```

We need to exec into the sharelatex container and prepare to install our dependencies:
```bash
docker exec -it sharelatex /bin/bash
apt update
# Check the currently installed schemes
tlmgr info schemes

# Check the currently installed collections
tlmgr info collections

# List the collections included in a scheme
tlmgr info --list scheme-basic
# List the CTAN packages included in a collection
tlmgr info --list collection-basic
```

Now that you *hopefully* know which packages/collections/schemes you're interested in, we're ready to install them. Note that it's a fairly long install, presumably because everything is done sequentially.
```bash
# Start by getting synchronised with your current packages.
tlmgr update --self --all
# Install what you want. I want tikz and glossaries-extra right now.
tlmgr install collection-pictures collection-latexextra collection-bibtexextra
```

For reference, at the time of writing this post, TeXLive is about 6GB if you install everything, which may be excessive for your environment. Manually install collections if you would prefer to only have the packages that you care about. If you don't care, instead of what we installed above, you can install `scheme-full` to get everything.

`bib2gls`, used by glossaries-extra, requires java to be installed. So we just need to install the jre inside our container:
```bash
apt install -y default-jre
# Check if bib2gls works
/usr/local/texlive/2025/bin/x86_64-linux/bib2gls --version
```

Now that we're done, leave container unless you have some other dependencies to install, and we'll create a new tagged image from this container. If you're anything like me, you haven't really played around with the `docker commit` command before, but there's time in life for everything:
```bash
# This command even works while the container is running!
oldversion=$(cat config/version)
newversion="${oldversion}-RC0"

# Note that there is a regex in the overleaf repo that ensures the image to be loaded follows the format: \d+.\d+.\d+(-RC\d+)?(-with-texlive-full)
# Therefore, a valid name would be 6.0.1-RC0 if you wanted to use that convention
# Of course, you could modify this file, but then you would be out-of-tree.
docker commit -a "SickMcNugget <sickmcnugget@hishappyplace.com>" -m "Add custom latex packages" -p sharelatex sharelatex/sharelatex:${newversion}
docker images | grep sharelatex
```

# Using our new image all the time
That's the difficult work finished already. We just need to swap the image that overleaf-toolkit uses by modifying that `version` file:
```bash
# Check the currently running container version
docker ps | grep sharelatex
# mine was 6.0.1

# Swap the versions
echo "${newversion}" > config/version
bin/docker-compose down
bin/up -d

# Check that the correct version is being used
docker ps | grep sharelatex
# mine was 6.0.1-RC0
```

That should be it, you now have a sharelatex image with custom LaTeX packages ready to go.
