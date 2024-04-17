# Build script for NVGT using the scons build system.
# NVGT - NonVisual Gaming Toolkit (https://nvgt.gg)
# Copyright (c) 2022-2024 Sam Tupy
# license: zlib

import os, multiprocessing

# setup
env = Environment()
SConscript("build/upx_sconscript", exports = ["env"])
env.SetOption("num_jobs", multiprocessing.cpu_count())
if env["PLATFORM"] == "win32":
	SConscript("build/windev_sconscript", exports = ["env"])
	env.Append(CCFLAGS = ["/EHsc", "/J", "/std:c++17", "/GF", "/Zc:inline", "/O2"])
	env.Append(LINKFLAGS = ["/NOEXP", "/NOIMPLIB"], no_import_lib = 1)
	env.Append(LIBS = ["tolk", "enet", "angelscript64", "SDL2"])
	env.Append(LIBS = ["Kernel32", "User32", "imm32", "OneCoreUAP", "dinput8", "dxguid", "gdi32", "winspool", "shell32", "iphlpapi", "ole32", "oleaut32", "delayimp", "uuid", "comdlg32", "advapi32", "netapi32", "winmm", "version", "crypt32", "normaliz", "wldap32", "ws2_32"])
else:
	env.Append(CXXFLAGS = ["-fms-extensions", "-std=c++17", "-fpermissive", "-O2", "-Wno-narrowing", "-Wno-int-to-pointer-cast",  "-Wno-unused-result"], LIBS = ["m"])
if env["PLATFORM"] == "darwin":
	# homebrew paths, as well as paths for a folder called macosdev containing headers and pre-built libraries like bass and steam audio.
	env.Append(CPPPATH = ["/opt/homebrew/include", "#macosdev/include"], CCFLAGS = ["-mmacosx-version-min=14.0"], LIBPATH = ["/opt/homebrew/lib", "#macosdev/lib"], LIBS = ["angelscript", "enet", "SDL2"])
elif env["PLATFORM"] == "posix":
	# Same custom directory here accept called lindev for now, we enable the gold linker to silence seemingly pointless warnings about symbols in the bass libraries, we also strip the resulting binaries, and we add /usr/local/lib to the libpath because it seems we aren't finding libraries unless we do manually.
	env.Append(CPPPATH = ["/usr/local/include", "#lindev/include"], LIBPATH = ["/usr/local/lib", "#lindev/lib"], LINKFLAGS = ["-fuse-ld=gold", "-s"])
	# We must explicitly denote the static linkage for several libraries or else gcc will choose the dynamic ones.
	env.Append(LIBS = [":libangelscript.a", ":libenet.a", ":libSDL2.a"])
env.Append(CPPDEFINES = ["POCO_STATIC"])
env.Append(CPPPATH = ["#ASAddon/include", "#dep"], LIBPATH = ["#build/lib"])
VariantDir("build/obj_src", "src", duplicate = 0)

# plugins
plugin_env = env.Clone()
static_plugins = []
for s in Glob("plugin/*/_SConscript"):
	plugname = str(s).split(os.path.sep)[1]
	if ARGUMENTS.get(f"no_{plugname}_plugin", "0") == "1": continue
	plug = SConscript(s, variant_dir = f"build/obj_plugin/{plugname}", duplicate = 0, exports = {"env": plugin_env})
	if plug: static_plugins.append(plug)
env.Append(LIBS = static_plugins)

# nvgt itself
sources = Glob("build/obj_src/*.cpp")
env.Append(LIBS = [["PocoFoundationMT", "PocoJSONMT", "PocoNetMT", "PocoZipMT"] if env["PLATFORM"] == "win32" else ["PocoFoundation", "PocoJSON", "PocoNet", "PocoZip"], "phonon", "bass", "bass_fx", "bassmix", "SDL2main"])
env.Append(CPPDEFINES = ["NVGT_BUILDING", "NO_OBFUSCATE"], LIBS = ["ASAddon", "deps"])
if env["PLATFORM"] == "win32":
	env.Append(LINKFLAGS = ["/SUBSYSTEM:WINDOWS", "/OPT:REF", "/OPT:ICF", "/delayload:bass.dll", "/delayload:bass_fx.dll", "/delayload:bassmix.dll", "/delayload:phonon.dll", "/delayload:Tolk.dll"])
elif env["PLATFORM"] == "darwin":
	sources.append("build/obj_src/macos.mm")
	env["FRAMEWORKPREFIX"] = "-weak_framework"
	env.Append(FRAMEWORKS = ["CoreAudio",  "CoreFoundation", "CoreHaptics", "CoreVideo", "AudioToolbox", "AppKit", "IOKit", "Carbon", "Cocoa", "ForceFeedback", "GameController", "QuartzCore"])
	env.Append(LIBS = ["objc"])
	env.Append(LINKFLAGS = ["-Wl,-rpath,'.'", "-mmacosx-version-min=14.0"])
elif env["PLATFORM"] == "posix":
	env.Append(LINKFLAGS = ["-Wl,-rpath,'lib'"])
if ARGUMENTS.get("no_user", "0") == "0":
	if os.path.isfile("user/nvgt_config.h"):
		env.Append(CPPDEFINES = ["NVGT_USER_CONFIG"])
	if os.path.isfile("user/_SConscript"):
		SConscript("user/_SConscript", exports = {"plugin_env": plugin_env, "nvgt_env": env})
SConscript("ASAddon/_SConscript", variant_dir = "build/obj_ASAddon", duplicate = 0, exports = "env")
SConscript("dep/_SConscript", variant_dir = "build/obj_dep", duplicate = 0, exports = "env")
env.Program("release/nvgt", sources)

# stubs
def fix_windows_stub(target, source, env):
	"""On windows, we replace the first 2 bytes of a stub with 'NV' to stop some sort of antivirus scan upon script compile that makes it take a bit longer."""
	for t in target:
		with open(str(t), "rb+") as f:
			f.seek(0)
			f.write(b"NV")
			f.close()
if ARGUMENTS.get("no_stubs", "0") == "0":
	stub_platform = "" # This detection will likely need to be improved as we get more platforms working.
	if env["PLATFORM"] == "win32": stub_platform = "windows"
	elif env["PLATFORM"] == "darwin": stub_platform = "mac"
	elif env["PLATFORM"] == "posix": stub_platform = "linux"
	else: stub_platform = env["PLATFORM"]
	VariantDir("build/obj_stub", "src", duplicate = 0)
	stub_env = env.Clone(PROGSUFFIX = ".bin")
	stub_env.Append(CPPDEFINES = ["NVGT_STUB"])
	if ARGUMENTS.get("stub_obfuscation", "0") == "1": stub_env["CPPDEFINES"].remove("NO_OBFUSCATE")
	# Todo: Can we make this use one list of sources E. the sources variable above? This scons issue where one must provide the variant_dir when specifying sources is why we are not right now.
	stub_objects = stub_env.Object(Glob("build/obj_stub/*.cpp"))
	if env["PLATFORM"] == "darwin":
		stub_objects.append(stub_env.Object("build/obj_stub/macos.mm"))
	stub = stub_env.Program(f"release/nvgt_{stub_platform}", stub_objects)
	if "upx" in env:
		stub_u = stub_env.UPX(f"release/nvgt_{stub_platform}_upx.bin", stub)
	# on windows, we should have a version of the Angelscript library without the compiler, allowing for slightly smaller executables.
	if env["PLATFORM"] == "win32":
		stub_env.AddPostAction(stub, fix_windows_stub)
		if "upx" in env: stub_env.AddPostAction(stub_u, fix_windows_stub)
		stublibs = list(stub_env["LIBS"])
		if "angelscript64" in stublibs:
			stublibs.remove("angelscript64")
			stublibs.append("angelscript64nc")
			stub_nc = stub_env.Program(f"release/nvgt_{stub_platform}_nc", stub_objects, LIBS = stublibs)
			stub_env.AddPostAction(stub_nc, fix_windows_stub)
			if "upx" in env:
				stub_nc_u = stub_env.UPX(f"release/nvgt_{stub_platform}_nc_upx.bin", stub_nc)
				stub_env.AddPostAction(stub_nc_u, fix_windows_stub)
