<div align="center">
    <p>
        <a href="https://latte.to/#gh-dark-mode-only" target="_blank">
            <img height="100" src="https://github.com/latte-soft/.github/raw/master/assets/latte-banner/latte-banner-dark-theme.svg#gh-dark-mode-only" />
        </a>
        <a href="https://latte.to/#gh-light-mode-only" target="_blank">
            <img height="100" src="https://github.com/latte-soft/.github/raw/master/assets/latte-banner/latte-banner-light-theme.svg#gh-light-mode-only" />
        </a>
        <br /> <br />
        <!-- DARK MODE -->
        <a href="https://latte.to/#gh-dark-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/dark-mode/website-icon-white.svg#gh-dark-mode-only" />
        </a>
        <a href="https://latte.to/invite/#gh-dark-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/dark-mode/discord-clyde-icon-white.svg#gh-dark-mode-only" />
        </a>
        <a href="https://twitter.com/lattesoftworks/#gh-dark-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/dark-mode/twitter-icon-white.svg#gh-dark-mode-only" />
        </a>
        <a href="https://github.com/latte-soft/#gh-dark-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/dark-mode/github-icon-white.svg#gh-dark-mode-only" />
        </a>
        <a href="https://www.roblox.com/groups/10685936/Latte-Softworks#!/about/#gh-dark-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/dark-mode/roblox-icon-white.svg#gh-dark-mode-only" />
        </a>
        <a href="mailto:support@latte.to" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/dark-mode/email-icon-white.svg#gh-dark-mode-only" />
        </a>
        <!-- LIGHT MODE -->
        <a href="https://latte.to/#gh-light-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/light-mode/website-icon-black.svg#gh-light-mode-only" />
        </a>
        <a href="https://latte.to/invite/#gh-light-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/light-mode/discord-clyde-icon-black.svg#gh-light-mode-only" />
        </a>
        <a href="https://twitter.com/lattesoftworks/#gh-light-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/light-mode/twitter-icon-black.svg#gh-light-mode-only" />
        </a>
        <a href="https://github.com/latte-soft/#gh-light-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/light-mode/github-icon-black.svg#gh-light-mode-only" />
        </a>
        <a href="https://www.roblox.com/groups/10685936/Latte-Softworks#!/about/#gh-light-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/light-mode/roblox-icon-black.svg#gh-light-mode-only" />
        </a>
        <a href="mailto:support@latte.to#gh-light-mode-only" target="_blank">
            <img width="24" src="https://github.com/latte-soft/.github/raw/master/assets/icons/light-mode/email-icon-black.svg#gh-light-mode-only" />
        </a>
    </p>
</div>

___

# 0x1D
Roblox Studio Remote Code Execution (RCE) Vulnerability | By [Latte Softworks](https://latte.to)

___

## Introduction
<sup><i>Documents & code snippets are licensed under the MIT License, see LICENSE.txt for more information.</i>
<br />
Written by <a href="https://github.com/regginator">Reggie</a> for <a href="https://latte.to">Latte Softworks</a>
</sup>

Hey, folks!
In late August this year, I discovered something fairly *peculiar* about the Roblox Binary Model Format (`rbxm`/`rbxl`) and a certain jump in `TypeId`s. (Each TypeId denotes a type of property in the binary format; e.g. `String`, `UDim`, `Vector3`, etc.. We'll learn more about this later.)
<br />
We are leaking this RCE due to the fact that Roblox has put out a change in `sitetest2.robloxlabs`/`zintegration` that could prevent direct bytecode execution through this method in the future. We'll talk about this part later.. ;)
<br /> <br />
I'll continue to refer to this binary format as the "`rbxm`" format, but I'ts the exact same as `rbxl`.
<br /> <br />
For me to be able to properly explain all of this, you will need to take in and understand a few key aspects of the format relevant to this "vulnarability":
<br />

- There are a couple of fairly well-made documents containing decent previous community findings and documentation on the rbxm format, the best probably being in [the docs for rbx-dom](https://dom.rojo.space/binary). (Yes, the same team that made [Rojo](https://rojo.space)) There are some minor issues with this doc such as the header signature being "`89 FF 0A 1A 0A`" (Hex), when it's actually `89 FF 0D 0A 1A 0A`.

- Each rbxm file contains first a [file header](https://dom.rojo.space/binary#file-header), and then a proceeding list of "[chunks](https://dom.rojo.space/binary#chunks)". We're personally interested in the [`PROP`](https://dom.rojo.space/binary#prop-chunk) and (more recently, undocumented) `SIGN` chunck.

- Chunks themselves *can* be compressed using either the [LZ4](https://en.wikipedia.org/wiki/LZ4_(compression_algorithm)) compression algorithm, or, (just this week, also undocumented) the ["zstd"](https://github.com/facebook/zstd) compression algorithm.<br />You can see more about how chunks with compression work [here](https://dom.rojo.space/binary#chunks).

- All [`PROP`](https://dom.rojo.space/binary#prop-chunk) chunks contain the Roblox property info itself, as-well as an enumerated 1-byte integer value we call the "`TypeId`". This TypeId references a very specific type of property in the format specifically so the serailizer can encode/decode each property. For example, `BaseScript.Source` is usually represented under the type id 1/0x01 (`String`). The first 4 bytes contain the specific length of bytes to read, the rest is the string itself. You can look at more examples [here](https://dom.rojo.space/binary#data-types).

- The `SIGN` chunk that was just recently added to Studio contains a "key"/hash of the file itself, used internally for loading "optimized" `CoreScriptPackage`s or `BuiltInPlugins`. As of now, we don't really have a clear workaround for this chunk. (It's REQUIRED for loading bytecode directly under TypeId 0x1D, which we'll talk about in just a moment)

Theres FAR MORE to the format to this in general, but this is really all you need to know for this situation. This vast information is also why we'll be working on our own official documentation of the modern rbxm format in the future, so stay tuned into our [Discord Server](https://latte.to/invite) for more!

## TypeId "`0x1D`"
We managed to discover this unique TypeId due to some official model files Roblox put out for their "`BuiltInPlugins`". It's read almost equivalent to the `String` property-type, except for the fact that it's binary data and not UTF-8. In the disassembled source, it is officially denoted as "`Bytecode`", so we're calling it `BytecodeString` in our pseudo reference!
<br />
In our *very* extensive testing, (so much we almost decided to jump off of a bridge) we figured out that bytecode wouldn't directly load in the Player, or RCC (Server), and would return a bad/corrupted file error or similar. This is ultimately due to the `SIGN` chunk Roblox uses internally in the rbxm format.
<br />
I'm only here talking about this in the first place because Roblox Studio doesn't actually totally respect this chunk for the type; right now, that is.. You can directly load Luau VM bytecode in studio under a developer plugin or model, no-matter if it's a local plugin or on the marketplace. :)
<br /> <br />
With that knowledge, you can directly load and run any Luau VM bytecode instantly under anyone's environment! In-case you didn't know, [that's almost like giving full machine-instruction access](https://v3rmillion.net/showthread.php?tid=1149589) lol.

## Luau Bytecode & RCE From the VM
I'm not *the best* when it comes to exploiting the Lua/Luau VM at runtime (modifying instructions directly), but you can use any number of these resources to achieve direct RCE in the Luau Virtual Machine:

- [Exploiting Lua 5.1 on x86_64](https://gist.github.com/ulidtko/51b8671260db79da64d193e41d7e7d16) (Still works the same way in [Luau](https://github.com/Roblox/luau/blob/master/VM/src/lvmexecute.cpp#L394))
- [Reply From Defcon42 on a Previous Exploit ACE Vulnarability](https://v3rmillion.net/showthread.php?pid=8114661#pid8114661) (Still can reference here)
- [Lua Bytecode Explorer](https://www.luac.nl/) (Similar to godbolt, but for Lua. Again, you can use these same concepts with the Luau VM)

## Proof of Concept
I've provided a basic proof-of-concept for this in the [src/build](src/build) directory, which when placed in your Studio `Plugins` folder and a game is loaded, will run directly under the Luau Bytecode Format V3. (Currently in Studio at the time of writing)
<br /> <br />
Since I'm LAZY (lol) I used [Rojo](https://rojo.space) to generate an RBXM with a script containing Luau bytecode, then editing the file in a hex editor afterward and changing the `TypeId` for `Script.Source` from 0x01 to 0x1D. Again, It's just meant to be a very basic proof of concept showing the direct loading of Luau bytecode from an rbxm.

## Conclusion
After gatekeeping this for a little over 2 months successfully without anyone else directly figuring out about this issue, we've decided to release a basic form of the method after Roblox has decided to add a change to how they handle this.
<br /> <br />
We will be releasing our own RBXM format implementation w/ DOM, which will also include further documentation on the format in the near future. Make sure to join [our Discord Server](https://latte.to) to stay tuned!

## Credits
- Reggie - [GitHub](https://github.com/regginator) | [V3rm](https://v3rmillion.net/member.php?action=profile&uid=1763716)
- LuaPhysics - [GitHub](https://github.com/LuaPhysics) | [V3rm](https://v3rmillion.net/member.php?action=profile&uid=1149709)