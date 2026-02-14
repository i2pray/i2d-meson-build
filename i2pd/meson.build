project('i2pd', 'cpp',
  version: 'vPray',
  license: 'MIT',
  default_options: [
    'cpp_std=c++20',
    'warning_level=3',
    'werror=false',
    'debug=true',
    'optimization=0'
  ],
  meson_version: '>= 1.2.0'
)

win_mod = import('windows')

cpp = meson.get_compiler('cpp')
cpp_id = cpp.get_id()
cpp_ver = cpp.version()
host_sys = host_machine.system()

# compiler sanity
if cpp_id == 'gcc'
  if cpp_ver.version_compare('<8.0')
    error('GCC too old; requires at least 8.0')
  endif
  add_project_arguments('-Wno-psabi', language: 'cpp')
elif cpp_id == 'clang'
  if cpp_ver.version_compare('<16.0')
    error('Clang too old')
  endif
  add_project_arguments(
    '-Wno-misleading-indentation',
    '-Wno-overloaded-virtual',
    language: 'cpp'
  )
endif

add_project_arguments(
  '-Wno-deprecated-declarations',
  '-Wno-unused-parameter',
  '-Wno-pedantic',
  '-Wno-type-limits',
  language: 'cpp'
)
add_project_arguments('-Wno-misleading-indentation', language: 'cpp')

if host_sys != 'windows'
  add_project_arguments('-fPIC', language: 'cpp')
endif

use_static = false
use_upnp = false
use_git_version = true
use_winxp_flags = false
use_win32_app = host_sys == 'windows'
use_aslr = false

if use_git_version
  git = find_program('git', required: true)
  git_version = run_command(git, 'describe', '--tags', check: true).stdout().strip()
  add_project_arguments('-DGITVER=' + git_version, language: 'cpp')
endif

if use_aslr
  add_project_link_arguments(
    '-Wl,--nxcompat',
    '-Wl,--dynamicbase',
    '-Wl,--high-entropy-va',
    language: 'cpp'
  )
endif

if host_sys == 'windows'
  add_project_link_arguments(
    '-static',
    '-static-libgcc',
    '-static-libstdc++',
    '-Wl,-Bstatic',
    '-lstdc++',
    language: 'cpp'
  )
endif

inc = include_directories(
  'libi2pd',
  'libi2pd_client',
  'i18n',
  'daemon',
  'libi2pd_wrapper',
  'Win32'
)


python = find_program(['python', 'python3'], required: true)

# libi2pd
lib_src_cmd = run_command(python, '-c', '''import os
print('\n'.join(
    os.path.join('libi2pd', f)
    for f in os.listdir('libi2pd')
    if f.endswith('.cpp')
))
''', check: true)

lib_src = lib_src_cmd.stdout().strip().splitlines()

# libi2pd_client
lib_client_src_cmd = run_command(python, '-c', '''import os
print('\n'.join(
    os.path.join('libi2pd_client', f)
    for f in os.listdir('libi2pd_client')
    if f.endswith('.cpp')
))
''', check: true)

lib_client_src = lib_client_src_cmd.stdout().strip().splitlines()

# i18n
lang_src_cmd = run_command(python, '-c', '''import os
print('\n'.join(
    os.path.join('i18n', f)
    for f in os.listdir('i18n')
    if f.endswith('.cpp')
))
''', check: true)

lang_src = lang_src_cmd.stdout().strip().splitlines()

# libi2pd_wrapper
wrap_lib_src_cmd = run_command(python, '-c', '''import os
print('\n'.join(
    os.path.join('libi2pd_wrapper', f)
    for f in os.listdir('libi2pd_wrapper')
    if f.endswith('.cpp')
))
''', check: true)

wrap_lib_src = wrap_lib_src_cmd.stdout().strip().splitlines()

# daemon (excluding UnixDaemon.cpp)
daemon_src_cmd = run_command(python, '-c', '''import os
print('\n'.join(
    os.path.join('daemon', f)
    for f in os.listdir('daemon')
    if f.endswith('.cpp') and f != 'UnixDaemon.cpp'
))
''', check: true)

daemon_src = daemon_src_cmd.stdout().strip().splitlines()

if host_sys == 'windows'
  win32_src_cmd = run_command(python, '-c', '''import os
print('\n'.join(os.path.join('Win32', f) for f in os.listdir('Win32') if f.endswith('.cpp')))
''', check: true)

  daemon_src += win32_src_cmd.stdout().strip().splitlines()

  add_project_arguments('-DWIN32_LEAN_AND_MEAN', language: 'cpp')

  if use_winxp_flags
    add_project_arguments(
      '-DWINVER=0x0501',
      '-D_WIN32_WINNT=0x0501',
      language: 'cpp'
    )
  endif

  if use_win32_app
    add_project_arguments('-DWIN32_APP', language: 'cpp')
  endif

else
  daemon_src += ['daemon/UnixDaemon.cpp']

  if host_sys == 'linux'
    add_project_arguments('-D_DEFAULT_SOURCE', '-D_GNU_SOURCE', language: 'cpp')
  endif
endif

# dependencies (force static on Windows)
if host_sys == 'windows'
  boost_dep = dependency('boost',
    modules: ['system', 'filesystem', 'program_options'],
    static: true,
    required: true
  )
  openssl_dep = dependency('openssl', static: true, required: true)
  zlib_dep = dependency('zlib', static: true, required: true)
else
  boost_dep = dependency('boost',
    modules: ['system', 'filesystem', 'program_options'],
    static: use_static,
    required: true
  )
  openssl_dep = dependency('openssl', static: use_static, required: true)
  zlib_dep = dependency('zlib', static: use_static, required: true)
endif

threads_dep = dependency('threads', required: true)

platform_deps = []

if host_sys == 'windows'
  platform_deps += [
    cpp.find_library('ws2_32'),
    cpp.find_library('iphlpapi'),
    cpp.find_library('crypt32'),
    cpp.find_library('gdi32'),
    cpp.find_library('ole32'),
    cpp.find_library('uuid'),
    cpp.find_library('mswsock'),
    cpp.find_library('winpthread'),
  ]
  daemon_res = win_mod.compile_resources(
    'Win32/Resource.rc',
    include_directories: inc
  )
  daemon_src += daemon_res
elif host_sys == 'linux'
  atomic_dep = cpp.find_library('atomic', required: false)
  dl_dep = cpp.find_library('dl', required: false)
  platform_deps += [atomic_dep, dl_dep]
endif

common_deps = [
  boost_dep,
  openssl_dep,
  zlib_dep,
  threads_dep
] + platform_deps

# static internal libraries (always)
libi2pd = static_library('i2pd', lib_src,
  include_directories: inc,
  dependencies: common_deps,
  install: true
)

libi2pdlang = static_library('i2pdlang', lang_src,
  include_directories: inc,
  dependencies: common_deps,
  install: true
)

libi2pdclient = static_library('i2pdclient', lib_client_src,
  include_directories: inc,
  dependencies: common_deps,
  link_with: [libi2pd, libi2pdlang],
  install: true
)

libi2pdwrapper = static_library('i2pdwrapper', wrap_lib_src,
  include_directories: inc,
  dependencies: common_deps,
  install: true
)

# shared libraries only on non-win
if host_sys != 'windows'
  shared_library('i2pd', lib_src,
    include_directories: inc,
    dependencies: common_deps,
    install: true
  )
endif

if host_sys == 'windows' and use_win32_app
  executable('i2pd', daemon_src,
    include_directories: inc,
    dependencies: common_deps,
    link_with: [libi2pd, libi2pdclient, libi2pdlang],
    win_subsystem: 'windows',
    install: true
  )
else
  executable('i2pd', daemon_src,
    include_directories: inc,
    dependencies: common_deps,
    link_with: [libi2pd, libi2pdclient, libi2pdlang],
    install: true
  )
endif
