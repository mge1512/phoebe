prog_python = import('python').find_installation(
  'python3',
  required : get_option('collector'),
  modules : ['cffi'],
  disabler : true,
)
collector_script = files('collect_stats.py')

abi_src = custom_target(
  '_phoebe.c',
  input : [collector_script],
  output : ['_phoebe.c'],
  command : [prog_python, '@INPUT@', 'generate', '@OUTPUT@'],
)
abi = shared_library(
  '_phoebe.abi3',
  abi_src, stat_src, common_src,
  name_prefix : '',
  dependencies : [prog_python.dependency(embed : true), common_dep, nl_nf_3],
)
