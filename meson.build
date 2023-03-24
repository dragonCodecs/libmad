# SPDX-FileCopyrightText: 2023 Amyspark <amy@amyspark.me>
# SPDX-License-Identifier: BSD-3-Clause

project(
    'mad',
    'c',
    version: '0.16.3',
    license: 'GPL-2.0-or-later',
    default_options: [
        'c_std=c99',
    ]
)

mad_args = []
cdata = configuration_data()

if get_option('optimize') == 'speed'
    message('Optimizing for speed over accuracy.')
    mad_args += '-DOPT_SPEED'
else
    message('Optimizing for accuracy over speed.')
    mad_args += '-DOPT_ACCURACY'
endif

compiler = meson.get_compiler('c')
system_processor = target_machine.cpu()

mad_srcs = files(
    'bit.c',
    'decoder.c',
    'fixed.c',
    'frame.c',
    'huffman.c',
    'layer12.c',
    'layer3.c',
    'stream.c',
    'synth.c',
    'timer.c',
    'version.c',
)

aso = get_option('aso')
fixed_point = get_option('fixed_point')
if compiler.sizeof('void*') == 8
    message('Using 64 bit fixed point math')
    cdata.set10('FPM_64BIT', fixed_point)
elif system_processor == 'i386'
    message('Using x86 fixed point math')
    cdata.set10('FPM_INTEL', fixed_point)
    if aso
        mad_args += '-DASO_ZEROCHECK'
    endif
elif system_processor == 'arm'
    message('Using ARM fixed point math')
    cdata.set10('FPM_ARM', fixed_point)
    if aso
        mad_args += ['-DASO_INTERLEAVE1', '-DASO_IMDCT']
        mad_srcs += files('imdct_l_arm.S')
    endif
elif system_processor == 'mips'
    message('Using MIPS fixed point math')
    cdata.set10('FPM_MIPS', fixed_point)
    if aso
        mad_args += ['-DASO_INTERLEAVE2', '-DASO_ZEROCHECK']
    endif
elif system_processor == 'sparc'
    message('Using SPARC fixed point math')
    cdata.set10('FPM_SPARC', fixed_point)
elif system_processor == 'ppc'
    message('Using PowerPC fixed point math')
    cdata.set10('FPM_PPC', fixed_point)
else
    warn('Target CPU architecture not detected. Fixed-point math will yield limited accuracy.')
    cdata.set10('FPM_DEFAULT', fixed_point)
endif

cdata.set('SIZEOF_INT', compiler.sizeof('int'))

cdata.set('CMAKE_PROJECT_VERSION_MAJOR', meson.project_version().split('.')[0])
cdata.set('CMAKE_PROJECT_VERSION_MINOR', meson.project_version().split('.')[1])
cdata.set('CMAKE_PROJECT_VERSION_PATCH', meson.project_version().split('.')[2])

mad_h = configure_file(
    input: 'mad.h.in',
    output: 'mad.h',
    configuration : cdata,
    format: 'cmake@'
)

mad_srcs += mad_h

header_macros = [
    ['sys/types.h', 'HAVE_SYS_TYPES_H'],
    ['sys/wait.h', 'HAVE_SYS_WAIT_H'],
    ['sys/mman.h', 'HAVE_SYS_MMAN_H'],
    ['sys/stat.h', 'HAVE_SYS_STAT_H'],
    ['unistd.h', 'HAVE_UNISTD_H'],
    ['assert.h', 'HAVE_ASSERT_H'],
    ['fcntl.h', 'HAVE_FCNTL_H'],
    ['limits.h', 'HAVE_LIMITS_H'],
]

foreach header : header_macros
    if compiler.has_header(header[0])
        mad_args += '-D@0@'.format(header[1])
    endif
endforeach

function_macros = [
    ['ftruncate', 'HAVE_FTRUNCATE'],
    ['pipe', 'HAVE_PIPE'],
    ['fork', 'HAVE_FORK'],
    ['waitpid', 'HAVE_WAITPID'],
    ['ftruncate', 'HAVE_FTRUNCATE'],
]

foreach function : function_macros
    if compiler.has_function(function[0])
        mad_args += '-D@0@'.format(function[1])
    endif
endforeach

mad_tmp = static_library(
    'mad_tmp',
    mad_srcs,
    c_args: mad_args,
)

makedef = find_program('packaging/makedef.py')

makedef_args = [makedef]

if compiler.get_argument_syntax() == 'msvc'
  dumpbin = find_program('dumpbin', required: true)
  makedef_args += ['--dumpbin', dumpbin]
else
  nm = find_program('nm', 'llvm-nm', required: true)
  makedef_args += ['--nm', nm]
endif

extern_prefix = compiler.symbols_have_underscore_prefix() ? '_' : ''
if target_machine.system() == 'windows'
    makedef_args += ['--prefix', extern_prefix]
    makedef_args += ['--format', 'msvc']
elif target_machine.system() == 'darwin'
    makedef_args += ['--format', 'darwin']
else
    makedef_args += ['--prefix', extern_prefix]
    makedef_args += ['--format', 'gcc']
endif

vs_module_def = custom_target(
    'mad_def',
    output: 'libmad.def',
    command: [makedef_args, mad_tmp],
    capture: true
)

mad_link_deps = []
mad_link_args = []
if target_machine.system() == 'darwin'
    mad_link_args += '-Wl,-exported_symbols_list,@0@'.format(meson.current_build_dir() / 'libmad.def')
    mad_link_deps += vs_module_def
elif target_machine.system() != 'windows' and target_machine.system() != 'cygwin'
    mad_link_args += '-Wl,--version-script=@0@'.format(meson.current_build_dir() / 'libmad.def')
    mad_link_deps += vs_module_def
endif

mad = library(
    'mad',
    link_whole: mad_tmp,
    link_args: mad_link_args,
    link_depends: mad_link_deps,
    vs_module_defs: vs_module_def,
    version: meson.project_version(),
    soversion: meson.project_version(),
    install: true,
)

install_headers(mad_h)

mad_dep = declare_dependency(
    link_with: mad,
    include_directories: include_directories('.')
)

meson.override_dependency('mad', mad_dep)

if get_option('example') and cdata.has('HAVE_UNISTD_H') and cdata.has('HAVE_SYS_STAT_H') and cdata.has('HAVE_SYS_MMAN_H')
    mad_example_srcs = files(
        'minimad.c',
    )

    mad_example = executable(
        'mad_example',
        mad_example_srcs,
        dependencies: mad_dep,
    )
endif

pkg = import('pkgconfig')

pkg.generate(
    mad,
    name: 'libmad',
    description: 'ID3 tag manipulation library for MP3 files',
    url: 'https://github.com/tenacityteam/libmad',
)
