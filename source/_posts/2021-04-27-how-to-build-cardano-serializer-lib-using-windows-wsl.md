title: >-
  How to build cardano-serializer-lib using Windows WSL
date: 2021-04-27 18:01:43
tags:
  - cardano
  - wsl
  - debian
categories:
  - Cardano
---

Follow these instructions to successfully build
[cardano-serialization-lib](https://github.com/Emurgo/cardano-serialization-lib)
on Windows (because using e.g. Git Bash will simply not work).

> This assumes you already have [Debian](https://aka.ms/wslstore)
> running on the
> [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/install-win10)

1. install python 2.7

	```bash
	sudo apt-get install python2.7
	```

2. install curl

	```bash
	sudo apt-get install curl
	```

2. install rust and **choose option 1** (Proceed with installation (default))

	```bash
	curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
	```
	
4. install nvm

	```bash
	curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
	```

5. install cmake

	```bash
	sudo apt-get install cmake
	sudo apt-get install build-essential
	```

6. install wasm-pack

	```bash
	curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
	```

7. restart your bash

	```bash
	source ~/.bashrc
	```

8. build the repository

	```bash
	git clone git@github.com:Emurgo/cardano-serialization-lib.git
	cd cardano-serialization-lib
	git submodule update --init --recursive
	nvm install v12.18.1
	nvm use
	npm install
	npm run rust:build-nodejs
	```

All done, you should now find your ready-to-use libraries inside folder `rust/pkg`.