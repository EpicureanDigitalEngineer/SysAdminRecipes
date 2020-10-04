# SysAdminRecipes

*System Administration Recipes* is a collection of recipes (HowTos, instructables, cheat sheets, etc.) for installing, maintaining and operating digital services. The recipes may prove interesting for system administrators who may in turn wish to help those recipes stay up-to-date and evolve.

The collection of documents is written in markdown and organised for use with `mkdocs`. The repository holds the source markdown files. Documents are organised in topics. For the time being, only `freebsd` is available - offering base recipes required to run any digital services. Future topics may include `dc-in-a-box` (running a data centre on a single box using a FreeBSD hypervisor (ala Proxmox)); home automation with Home Assistant; digital management of photos/ebooks/documents.

:warning: **WARNING**:  System Administration is the noble art of making computers work. And System Administrators have the responsibility when things don't work (which may be more often than we care to admit). While the recipes have all been tested as advertised, your mileage may vary when applying them. Make sure you know what you are doing when following any recipe as executing root-level commands may lead to catastrophic failure for your systems.

[Read the recipes](https://epicureandigitalengineer.github.io/SysAdminRecipes/)

## Project layout

    mkdocs.yml    	            # The configuration file.
    docs/
        index.md             	# The documentation homepage.
        img/					# Images to be included in the documentation
        topic/index.md		    # Introduction page for the topic
        topic/recipe1.md		# A recipe about the topic
## Licence

Copyright (C) The Epicurean Digital Engineer 

These pages are licenced under a [Creative Commons Attribution Non Commercial 4.0 international Licence](http://creativecommons.org/licenses/by-nc/4.0/). 


