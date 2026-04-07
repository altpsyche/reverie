<h1 align="center">Reverie</h1>

Reverie is a development environment focused on memory analysis and modification of games and applications for personal use. It is the successor to Cheat Engine, refactored to remove monetization, telemetry, and legacy OS support, and rebranded as a clean, professional, modern toolchain.


# Build Instructions

Reverie is built with Lazarus 4.x / FPC 3.2.2 (or later) on Windows 10+.

  1. Install Lazarus 4.x with FPC 3.2.2 (32-bit + 64-bit cross-compiler).
  2. Open `Cheat Engine/cheatengine.lpi` in the Lazarus IDE, or run:

         lazbuild --build-mode="Release 64-Bit" "Cheat Engine/cheatengine.lpi"

  3. Optionally compile the secondary projects you intend to use:

         speedhack.lpi          - 32/64-bit DLLs for speedhack capability
         luaclient.lpi          - 32/64-bit DLLs for {$luacode} capability
         vehdebug.lpi           - 32/64-bit DLLs for the VEH debugger interface
         DirectXMess.sln        - 32/64-bit D3D overlay and snapshot DLLs
         DotNetCompiler.sln     - cscompile Lua command support
         MonoDataCollector.sln  - 32/64-bit Mono inspection DLLs
         DotNetDataCollector.sln - 32/64-bit .NET symbol collectors
         tcclib.sln             - 32-32, 64-32, 64-64 builds for {$C} / {$CCODE}
         DBKKernel.sln          - kernel-mode driver (requires WDK + signing)

`.sln` files require Visual Studio 2017 or later. Build system unification under CMake is planned for Phase 4.
