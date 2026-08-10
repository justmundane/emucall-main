# emucall-main

`emucall` calls functions inside another process without injecting into it. It is a from-scratch x86-64 interpreter: it reads the target's own instruction bytes over `ReadProcessMemory` and executes them in your address space against a virtual CPU, a fake stack and a synthetic thread environment. No module is loaded into the target, no thread is hijacked, and nothing is written to it.

## How it works

A call sets up a virtual context and runs the target's code until it returns:

- **Stack** — a local buffer mapped at a fixed virtual base, with `rsp` seeded below the return address. Reads and writes that land in that range hit the buffer; everything else falls through to the real process.
- **Return detection** — the return address is a sentinel (`0xDEADBEEFDEADBEEF`); execution stops when `rip` reaches it, and the return value is read out of `rax` or `xmm0`.
- **Heap** — an emulated arena for anything the callee allocates, plus argument marshalling for managed, ANSI and wide strings so you can pass strings into functions that expect them.
- **Thread environment** — a synthetic TEB and 64 TLS slots, so code that touches thread-local state doesn't fault out.
- **External calls** — imports and runtime helpers that can't be sensibly interpreted are intercepted at their entry point and serviced by a handler instead of being stepped through.
- **Guard rails** — a per-call instruction limit (default 10,000) so a mispredicted branch spins out rather than hanging, and a ring buffer of the last 32 instructions that can be dumped when a call fails.

## Usage

Header-only. Add `emucall/` to your include path and include `source/imports.hpp`.

```cpp
if ( !memory->attach ( L"game.exe" ) )
{
    return 1;
}

std::vector< std::uint8_t > fake_stack ( 0x400000 );

auto cpu = emulator
(
    /* read  */ [ & ] ( std::uint64_t address, void* buffer, std::size_t size ) -> bool { ... },
    /* write */ [ & ] ( std::uint64_t address, const void* buffer, std::size_t size ) -> bool { ... }
);

caller->set_emulator ( &cpu );
caller->register_functions ( memory->game_assembly, memory->unity_player );

const auto result = caller->call< std::uint64_t > ( base + rva, argument );
```

The two callbacks decide where each access goes — the fake stack, the emulated heap, or the real process. `main.cpp` is a working skeleton of that wiring.

Instance methods take the `this` pointer as the first argument, and `extcall::injected` marks calls whose arguments are already staged in the emulated heap:

```cpp
caller->call< void > ( address, extcall::injected, this_ptr, argument );
```

## Coverage

The decoder and instruction handlers cover the general-purpose integer set, control flow, the x87 and SSE floating-point paths and part of AVX. Coverage is driven by what real target functions actually execute, so it is broad rather than complete — an unimplemented opcode fails the call and reports the instruction rather than silently returning garbage.

| | lines |
|---|---|
| decoder | 4,846 |
| floating point | 4,156 |
| arithmetic | 2,853 |
| call layer | 1,285 |
| emulator core | 1,306 |
| movement / flow / AVX | 1,221 |

## Target-specific values

There are none — no hardcoded addresses or build-specific RVAs anywhere in the library, so nothing goes stale when the target updates.

`register_functions` takes the `GameAssembly.dll` and `UnityPlayer.dll` bases and registers those ranges plus the Win32 and heap hooks, which orients the external-call layer around IL2CPP Unity targets. Everything it needs is resolved from those two bases at runtime. The emulator core itself has no such dependency.

## Building

Windows x64, C++20, MSVC. Open `emucall.slnx` and build Release|x64, or:

```
msbuild emucall.slnx /p:Configuration=Release /p:Platform=x64
```

`mem_lib` opens the target with `PROCESS_ALL_ACCESS` but only ever reads from it — no write path is exposed.

## License

MIT. See [LICENSE](LICENSE).
