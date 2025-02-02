# Rampastring.XNAUI v2.2.0 with IME backport

Warning: Please upgrade to the latest [xna-cncnet-client](https://github.com/CnCNet/xna-cncnet-client/) (version >= 2.11.0.0) instead of using this backport. It's not recommended to struggle with legacy xna-cncnet-client (version < 2.8.0.0).

## Steps to apply this backport to legacy xna-cncnet-client (version < 2.8.0.0)

1. Build or download the compiled files of this Rampastring.XNAUI from GitHub Release, and copy them to the Reference folder *accordingly*. Do not just unzip & override -- folder names differ.

2. For all C# projects in xna-cncnet-client solution, upgrade the .NET Framework requirement to at least .NET Framework 4.6 (including the XNA build). Migrate the project from `packages.config` into `PackageReference` (otherwise, you will regret when introducing ImeSharp).

3. For all C# projects in xna-cncnet-client solution, override the C# language version as at least C# 12.

4. In ClientGUI project, introduce the dependency `ImeSharp` using a conditional package reference:   
   
   ```xml
   <ItemGroup>
   <PackageReference Include="ImeSharp" Version="1.4.0" Condition="'$(Platform)' == 'SharpDX'" />
   </ItemGroup>
   ```

5. Create `IME\IMEHandler.cs`, `IME\WinFormsIMEHandler.cs`, and `IME\DummyIMEHandler.cs` files in ClientGUI project, and include them in the project file:   
   
   ```xml
   <ItemGroup>
   <Compile Include="IME\DummyIMEHandler.cs" />
   <Compile Include="IME\IMEHandler.cs" />
   <Compile Include="IME\WinFormsIMEHandler.cs" Condition="'$(Platform)' == 'SharpDX'" />
   </ItemGroup>
   ```

6. Copy the content of these three files:
- [IMEHandler.cs](https://github.com/CnCNet/xna-cncnet-client/blob/7c02b0dea97e0ee2deecc9278a68401fb14dd054/ClientGUI/IME/IMEHandler.cs)

- [WinFormsIMEHandler.cs](https://github.com/CnCNet/xna-cncnet-client/blob/7c02b0dea97e0ee2deecc9278a68401fb14dd054/ClientGUI/IME/WinFormsIMEHandler.cs)

- [DummyIMEHandler.cs](https://github.com/CnCNet/xna-cncnet-client/blob/7c02b0dea97e0ee2deecc9278a68401fb14dd054/ClientGUI/IME/DummyIMEHandler.cs)
7. Note that the compile-time constants differ between legacy xna-cncnet-client (version < 2.8.0.0) and the latest [xna-cncnet-client](https://github.com/CnCNet/xna-cncnet-client/) (version >= 2.11.0.0). Modify `IMEHandler.Create()` method:
   
   ```cs
    public static IMEHandler Create(Game game)
    {
   #if !XNA && !WINDOWSGL
        return new WinFormsIMEHandler(game);
   #else
        return new DummyIMEHandler();
   #endif
    }
   ```

8. Modify `DXMainClient\DXGUI\GameClass.cs` file to initialize `IMEHandler`:
   
   ```cs
   protected override void Initialize()
   {
      // Codes ...
      WindowManager wm = new WindowManager(this, graphics);
      wm.Initialize(content, ProgramConstants.GetBaseResourcePath());
      wm.IMEHandler = IMEHandler.Create(this); // Add this line
      // Codes ...    
   }
   ```

9. The legacy xna-cncnet-client (version < 2.8.0.0) requires manually editing the script to copy dll files, which is not straightforward compared with the latest [xna-cncnet-client](https://github.com/CnCNet/xna-cncnet-client/) (version >= 2.11.0.0). Modify `CopyCompiled.bat` file to copy these additional dependencies required by ImeSharp:
   
   ```bat
   echo Windows
   copy ImeSharp.dll %winBinaries%ImeSharp.dll
   copy TsfSharp.dll %winBinaries%TsfSharp.dll
   copy SharpGen.Runtime.dll %winBinaries%SharpGen.Runtime.dll
   copy SharpGen.Runtime.COM.dll %cr%SharpGen.Runtime.COM.dll
   copy System.Buffers.dll %winBinaries%System.Buffers.dll
   copy System.Memory.dll %winBinaries%System.Memory.dll
   copy Microsoft.Win32.Registry.dll %winBinaries%Microsoft.Win32.Registry.dll
   copy System.Runtime.CompilerServices.Unsafe.dll %winBinaries%System.Runtime.CompilerServices.Unsafe.dll
   copy System.Runtime.InteropServices.RuntimeInformation.dll %winBinaries%System.Runtime.InteropServices.RuntimeInformation.dll
   ```
   
   For unknown reasons, we can only place `SharpGen.Runtime.COM.dll` file to `Resources` folder in order to get the legacy xna-cncnet-client (version < 2.8.0.0) correctly work with IME. The latest [xna-cncnet-client](https://github.com/CnCNet/xna-cncnet-client/) (version >= 2.11.0.0) does not have this drawback.

10. The legacy xna-cncnet-client (version < 2.8.0.0) requires manually specifying DLL names to be loaded. Modify `DXMainClient\Program.cs` file:
    
    ```cs
    static List<string> SPECIFIC_LIBRARIES = new List<string>()
    {
      "ClientGUI",
      "ClientCore",
      "DTAConfig",
      "Localization",
      "MonoGame.Framework",
      "Rampastring.XNAUI",
      "Sdl",
      "soft_oal",
    
      // ImeSharp and it's dependencies
      "ImeSharp",
      "TsfSharp",
      "SharpGen.Runtime",
      "SharpGen.Runtime.COM",
      "System.Runtime.CompilerServices.Unsafe",
      "System.Buffers",
      "System.Memory",
      "Microsoft.Win32.Registry",
      "System.Runtime.InteropServices.RuntimeInformation",
    };
    ```

11. In addition, since these projects have been migrated into `PackageReference`, we need to modify the build scripts to some workaround some build failures.
    
    - `Build.bat` file:
    ```diff
    --- a/BuildScripts/Build.bat
    +++ b/BuildScripts/Build.bat
    echo Compiling %configuration% %platform%
    ECHO.
    +rd /q /s ..\ClientCore\obj >nul 2>&1
    +rd /q /s ..\ClientGUI\obj >nul 2>&1
    +rd /q /s ..\DTAConfig\obj >nul 2>&1
    +rd /q /s ..\DXMainClient\obj >nul 2>&1
    +dotnet restore ..\ClientGUI\ClientGUI.csproj
    "%msbuild%" ..\DXClient.sln /t:Rebuild /p:Platform=%platform% /p:Configuration=%configuration%
    if errorlevel 1 goto error
    ```

    - `ClientGUI.csproj` file:
    ```diff
    @@ -15,6 +15,10 @@
      </TargetFrameworkProfile>
      <NuGetPackageImportStamp>
      </NuGetPackageImportStamp>
    +    <RuntimeIdentifier Condition="'$(Platform)' == 'WindowsGL'">win-x86</RuntimeIdentifier>
    +    <RuntimeIdentifier Condition="'$(Platform)' != 'WindowsGL'">win</RuntimeIdentifier>
    +    <PlatformTarget Condition="'$(Platform)' == 'WindowsGL'">x86</PlatformTarget>
    +    <PlatformTarget Condition="'$(Platform)' != 'WindowsGL'">AnyCPU</PlatformTarget>
    </PropertyGroup>
    <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|SharpDX' ">
      <DebugSymbols>true</DebugSymbols>
    @@ -34,7 +38,6 @@
      <WarningLevel>4</WarningLevel>
    </PropertyGroup>
    <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|WindowsGL' ">
    -    <PlatformTarget>AnyCPU</PlatformTarget>
      <DebugSymbols>true</DebugSymbols>
      <DebugType>full</DebugType>
      <Optimize>false</Optimize>
    @@ -44,7 +47,6 @@
      <WarningLevel>4</WarningLevel>
    </PropertyGroup>
    <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|WindowsGL' ">
    -    <PlatformTarget>AnyCPU</PlatformTarget>
      <DebugType>pdbonly</DebugType>
      <Optimize>true</Optimize>
      <OutputPath>bin\WindowsGL\Release</OutputPath>
    @@ -53,7 +55,6 @@
      <WarningLevel>4</WarningLevel>
    </PropertyGroup>
    <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|XNAFramework' ">
    -    <PlatformTarget>x86</PlatformTarget>
      <DebugSymbols>true</DebugSymbols>
      <DebugType>full</DebugType>
      <Optimize>false</Optimize>
    @@ -63,7 +64,6 @@
      <WarningLevel>4</WarningLevel>
    </PropertyGroup>
    <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|XNAFramework' ">
    -    <PlatformTarget>x86</PlatformTarget>
      <DebugType>pdbonly</DebugType>
      <Optimize>true</Optimize>
      <OutputPath>bin\XNAFramework\Release</OutputPath>
    ```

12. Test whether IME works as expected. Note: if any of these dependent DLL files are missing, the client might either crash itself or just silently ignore the error.