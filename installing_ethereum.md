# Installing Ethereum

Currently stable clients are available for PoC-7 (Proof of Concept VII):
- [AlethZero, eth and neth](https://github.com/ethereum/cpp-ethereum/wiki) (C++)
- [Mist](https://github.com/ethereum/go-ethereum/releases) (Go) 
- [pyeth](https://github.com/ethereum/pyethereum/releases) (Python)

Bleeding edge PoC-8 code can be cloned from the develop branch of their git repositories:
- https://github.com/ethereum/cpp-ethereum
- https://github.com/ethereum/go-ethereum
- https://github.com/ethereum/pyethereum



homebrew-ethereum
=================

Homebrew Tap for Ethereum

## Installation

```
brew tap ethereum/ethereum
```

### C++ client
```
brew install ethereum
brew linkapps
```

### Go client
```
brew install go-ethereum
```

## Running

### C++ client
`open /Applications/AlethZero.app`, `eth` (CLI), or `neth` (ncurses interface)

### Go client
`mist` (GUI) or `ethereum` (CLI)


## Development
Get the latest development version with the `--devel` flag. Use `--build-from-source` if you don't want a pre-built bottle. Alternatively you can use the `--successful` flag (see [important note below](#important-note-when-using---successful)) or any other [available options](#options).

### C++ client
```
brew reinstall ethereum --devel
```

For the latest successful build on develop (last successful build from [cpt-obvious](http://build.ethdev.com/waterfall)):
```
brew reinstall ethereum --devel --successful
```

#### Important note when using --successful

If you get an error looking like this:
```
==> Cloning https://github.com/ethereum/cpp-ethereum.git
Updating /Library/Caches/Homebrew/ethereum--git
fatal: reference is not a tree: <latest commit hash>
Error: Failed to download resource "ethereum"
Failure while executing: git checkout -q -f
```

Either try `brew fetch ethereum --devel` or simply delete the cache with `rm -rf /Library/Caches/Homebrew/ethereum--git`


### Go client
```
brew reinstall eth-go go-ethereum --HEAD
```

Current branches (C++):
* `--HEAD` is on latest release branch [poc-7+](https://github.com/ethereum/cpp-ethereum/commits/poc-7+)
* `--devel` is on [develop](https://github.com/ethereum/cpp-ethereum/commits/develop)
* normal install is on master branch

Go:
* `--devel` is on develop branch
* normal install is on master branch


## Upgrading

```
brew update && brew upgrade
```

## Minor updates

### C++ client
```
brew update && brew reinstall ethereum
```

### Go client
```
brew update && brew reinstall eth-go go-ethereum
```

## Versions
List available versions with:
```
brew versions ethereum
```

If you have other versions installed, you can switch with:
```
brew switch ethereum <version>
```
Or follow this [StackOverflow answer](http://stackoverflow.com/a/9832084/2639784)

These brews can be installed via the raw GitHub URLs, or by cloning this
repository locally with `brew tap ethereum/ethereum`.

##Options

See `brew info ethereum` or `brew info go-ethereum` for all options. `--with-...` features are experimental patches.

Option               | desc.
---------------------|---------
`--headless`         | Headless
`--successful`       | Last successful build using --devel only
`--with-debug`       | Pass -DCMAKE_BUILD_TYPE=Debug
`--without-jsonrpc`  | Build without libjson-rpc-cpp as dependency
`--without-paranoia` | Build with -DPARANOIA=0

### jsonrpc

`brew info jsonrpc`

Option           | desc.
-----------------|---------
`--allow-origin` | Add Access-Control-Allow-Origin: * in headers

##Troubleshooting

* Use `--verbose` to get more info while installing.
* Make sure to update XCode and the command line tools.
* Run `brew update` and `brew upgrade`
* Fix what the `brew doctor` says.
* Reinstall dependencies: `brew reinstall boost --c++11 --with-python`
* Make changes to `/usr/local/Library/Taps/ethereum/homebrew-ethereum/ethereum.rb`
* Reinstall with `brew reinstall ethereum.rb` (send a pull request!)
* Take a walk

##Patching

First `cd /Library/Caches/Homebrew/ethereum--git/` and make your changes. Then `git diff > shiny.patch`, copy/paste the content of your patch under `__END__` of `ethereum.rb` and replace the `def patches` block with:

```
def patches
  DATA
end
```

If you want to submit your change, save your patch in a gist, add your `option 'shiny-option', 'Shiny description'` and the URL to your gist in the patches block and submit a pull request. Make sure to send a pull request to Ethereum also!
